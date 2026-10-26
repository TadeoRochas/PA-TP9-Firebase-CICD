# Práctica: CI/CD con Firebase Hosting y Hardening de Headers HTTP

**Asignatura:** Programación Avanzada (2026)  
**Institución:** FCyT - Licenciatura en Sistemas de Información  
**Alumno:** Tadeo Rochas  
**Repositorio GitHub:** [https://github.com/TadeoRochas/PA-TP9-Firebase-CICD](https://github.com/TadeoRochas/PA-TP9-Firebase-CICD)  
**URL de Producción:** [https://pa-tp9-firebase-cicd.web.app](https://pa-tp9-firebase-cicd.web.app)  

---

## 1. Salidas de Verificación de Headers (`check-headers.mjs`)

### 1.1 Medición Inicial en Producción (B1 - Sin Headers Configurados)
```text
▶ HTML: https://pa-tp9-firebase-cicd.web.app/
  status 200
  ✘ content-security-policy (falta)
  ✔ strict-transport-security: max-age=31556926; includeSubDomains; preload
  ✘ x-content-type-options (falta)
  ✘ x-frame-options (falta)
  ✘ referrer-policy (falta)
  ✘ permissions-policy (falta)
  · cache-control: max-age=3600

▶ Asset estático: /_next/static/chunks/1x6obpmlu1_c2.js
  ✘ cache-control: max-age=3600

❌ 6 control(es) pendiente(s).
```

---

## 2. Respuestas a las Preguntas de Reflexión

### Pregunta 1: ¿Por qué el preview tiene vencimiento? ¿Qué riesgos tendría un preview que no vence nunca?
Los canales de preview son **entornos efímeros** diseñados exclusivamente para validar cambios específicos introducidos en un Pull Request antes de su integración a la rama principal (`main`).

**Razones del vencimiento (`expires: 3d`):**
1. **Gestión de recursos y límites del proveedor:** Aunque Firebase Hosting ofrece una capa gratuita (Plan Spark), los canales permanentes consumen cuotas de almacenamiento de versiones y metadatos de hosting. La expiración automática previene la acumulación indefinida de canales huérfanos generados por ramas o PRs cerrados o abandonados.
2. **Superficie de ataque y riesgos de seguridad:** Un canal que no expira nunca deja versiones intermedias, potencialmente inestables, con dependencias desactualizadas o con vulnerabilidades de seguridad abiertas y accesibles públicamente a través de Internet bajo subdominios válidos de Firebase.
3. **Consistencia de datos y confusión de usuarios:** Previene que enlaces a previews viejos continúen circulando en chats, tickets o documentación interna, evitando que testers o clientes interactúen con versiones obsoletas creyendo que corresponden a producción.

---

### Pregunta 2: ¿Qué diferencia hay entre desplegar y liberar? ¿Dónde aparece cada concepto en este flujo?
- **Desplegar (Deploy):** Consiste en transferir, instalar y alojar los artefactos de software compilados (HTML, JS, CSS) en un entorno o servidor de infraestructura donde se encuentran disponibles y operativos para su ejecución.
- **Liberar (Release):** Consiste en exponer activamente esa versión del software a los usuarios finales de producción, haciéndola el punto de entrada oficial del servicio o producto.

**Dónde aparece cada concepto en nuestro flujo:**
- **Despliegue sin liberación:** Al abrir un Pull Request en GitHub, el workflow `Preview en Pull Request` **despliega** la compilación en un canal temporal (`preview channel`), pero **no la libera** al público; los usuarios que ingresan a `https://pa-tp9-firebase-cicd.web.app` siguen viendo la versión estable de producción.
- **Liberación:** Al hacer merge del PR hacia la rama `main`, el workflow `Deploy a producción` despliega y **libera** los cambios en el canal `live`.
- **Desacoplamiento mediante Rollback:** Cuando realizamos un rollback desde la consola de Firebase (`Release history -> Rollback`), no recompilamos ni volvemos a desplegar código: cambiamos el puntero de la versión **liberada** apuntando a una versión previa que ya había sido desplegada con anterioridad, restaurando el servicio en cuestión de segundos.

---

### Pregunta 3: ¿Por qué la clave de la service account va en un secreto y no en el repo? ¿Qué podría hacer alguien que la obtenga? ¿Cómo limitarías el daño?
La clave de la Service Account es un par de claves criptográficas en formato JSON que otorga identidad de máquina ante las APIs de Google Cloud y Firebase.

- **Por qué debe almacenarse en GitHub Secrets:** Un repositorio Git (especialmente público o compartido) tiene un historial inmutable y distribuido. Commitear credenciales en texto plano expone el proyecto a escaneos automáticos de bots maliciosos que rastrean claves en GitHub en milisegundos.
- **Qué podría hacer un atacante:**
  - Desplegar contenido malicioso, scripts de phishing o malware suplantando la identidad del sitio web legítimo.
  - Eliminar canales de hosting, degradar el servicio y causar denegación de servicio (DoS).
  - Si la cuenta tiene privilegios heredados o excesivos en GCP, podría acceder a otros recursos del proyecto (Firestore, Storage, Cloud Functions), extraer datos sensibles o crear instancias computacionales para minería de criptomonedas, derivando en compromisos graves o facturaciones imprevistas.
- **Cómo limitar el daño (Principio de Mínimo Privilegio y Buenas Prácticas):**
  1. **Roles mínimos (IAM):** Otorgar únicamente el rol `Firebase Hosting Admin` (`roles/firebasehosting.admin`), impidiendo el acceso a bases de datos, autenticación o facturación.
  2. **Rotación periódica y revocación inmediata:** Revocar la clave desde la consola de Google Cloud IAM ante la menor sospecha de compromiso.
  3. **Workload Identity Federation:** Como mejora arquitectónica avanzada, reemplazar el uso de claves JSON estáticas por federación de identidades OIDC entre GitHub Actions y Google Cloud, generando tokens temporales de vida corta sin requerir almacenar secretos de larga duración.

---

### Pregunta 4: La CSP rompió la app: ¿por qué? ¿Qué opciones evaluaste y qué costo de seguridad tiene cada una?
Al aplicar la política estricta inicial (`default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self';`):
1. **Fallo en scripts e interactividad (`/cursos`):** Next.js (en su modo de exportación estática) genera etiquetas `<script>` en línea (inline scripts) en el HTML para inicializar el runtime de Turbopack/webpack y el array de streaming de datos (`self.__next_f.push(...)`). Al tener `script-src 'self'`, el navegador bloquea la ejecución de estos scripts en línea por no provenir de un archivo `.js` externo. En consecuencia, el componente cliente `FiltroCursos.tsx` no se hidrata y los botones de filtro y el buscador quedan inoperativos.
2. **Fallo en imágenes externas (Home `/`):** La imagen decorativa de la portada proviene de `https://picsum.photos/seed/...`, la cual responde con una redirección HTTP 302 hacia la CDN de imágenes `https://fastly.picsum.photos/...`. Con `img-src 'self'`, el navegador bloqueó tanto la petición inicial como el recurso final redirigido.

**Opciones evaluadas y sus costos de seguridad:**
- **Opción A (Adoptada): Permitir `'unsafe-inline'` en `script-src` y `style-src`:**
  - *Ventaja:* Permite la hidratación completa de Next.js y los estilos en línea requeridos sin romper la compatibilidad estática.
  - *Costo de seguridad:* Abre una brecha para ataques de Cross-Site Scripting (XSS) si existiera alguna vulnerabilidad de inyección en la app, ya que el navegador ejecutará cualquier script en línea presente en el DOM.
- **Opción B (Nivel Destacado): Cálculo de hashes SHA-256 (`'sha256-...'`):**
  - *Ventaja:* Elimina `'unsafe-inline'`. El navegador solo ejecuta los scripts en línea cuyo hash coincida exactamente con los declarados en la cabecera.
  - *Implementación requerida:* En un entorno puramente estático (sin servidor SSR para emitir un `nonce` por request), los hashes deben calcularse en el pipeline de CI/CD inmediatamente después del paso `npm run build`, parseando los archivos HTML en `out/` e inyectando los hashes calculados en las cabeceras de `firebase.json` antes del despliegue.
- **Opción C: Dominios explícitos para imágenes:**
  - Se configuró específicamente `img-src 'self' https://picsum.photos https://fastly.picsum.photos data:`, evitando comodines inseguros como `img-src *` o `https:`, asegurando que solo el origen legítimo y su CDN provean imágenes.

---

### Pregunta 5: Para cada header de B2, ¿qué ataque mitiga?

| Cabecera HTTP | Valor Configurado | Ataque / Riesgo que Mitiga |
|---|---|---|
| **Strict-Transport-Security (HSTS)** | `max-age=63072000; includeSubDomains; preload` | Mitiga ataques de **Man-in-the-Middle (MitM)**, SSL Stripping y downgrade de protocolo, forzando a los navegadores a comunicarse exclusivamente mediante HTTPS durante los próximos 2 años, inclusive en subdominios. |
| **X-Content-Type-Options** | `nosniff` | Previene el **MIME-type Sniffing**. Evita que el navegador ignore el `Content-Type` declarado y ejecute archivos interpretándolos erróneamente (por ejemplo, tratar un archivo de texto o imagen subido por un usuario como si fuera ejecutable o JavaScript). |
| **X-Frame-Options** | `DENY` | Mitiga ataques de **Clickjacking** e intrusión de interfaz, prohibiendo taxativamente que la aplicación sea embebida dentro de elementos `<iframe>`, `<frame>`, `<embed>` u `<object>` de sitios de terceros. |
| **Referrer-Policy** | `strict-origin-when-cross-origin` | Mitiga la **fuga de información confidencial (Information Leakage)** en URLs. Al navegar dentro del mismo origen envía la URL completa, pero al navegar hacia dominios externos sólo envía el dominio/origen (eliminando paths o query params con tokens), y no envía nada si se pasa de HTTPS a HTTP. |
| **Permissions-Policy** | `camera=(), microphone=(), geolocation=(), browsing-topics=()` | Mitiga el **acceso no autorizado o silencioso a sensores y APIs sensibles del dispositivo** (cámara, micrófono, geolocalización, seguimiento de temas de navegación) por parte de scripts propios o bibliotecas de terceros embebidas. |
| **Cross-Origin-Opener-Policy (COOP)** | `same-origin` | Aísla el contexto de navegación del documento principal frente a ventanas emergentes externas, mitigando ataques de fuga de referencias cruzadas entre orígenes como **Spectre** y ataques de timing/side-channel. |

---

### Pregunta 6: ¿Por qué los assets de `/_next/static/` pueden cachearse un año y el HTML no?
- **Assets versionados (`/_next/static/**`):** Next.js utiliza una estrategia de compilación basada en **Content Hashing**. Cada archivo de JavaScript o CSS incluye en su nombre un hash criptográfico derivado de su contenido exacto (por ejemplo, `1x6obpmlu1_c2.js`). Si el código de un archivo cambia un solo caracter, el nombre del archivo resultante será completamente diferente. Por lo tanto, estos recursos son verdaderamente **inmutables**: un archivo con ese nombre jamás cambiará su contenido. Esto permite configurarlos con `Cache-Control: public, max-age=31536000, immutable`, eliminando peticiones al servidor y acelerando la navegación.
- **Archivos HTML (`out/*.html` o `/`):** El HTML es el punto de entrada que enlaza los nombres de los assets actuales. Si el navegador o la CDN cachearan el HTML por un período prolongado, los usuarios no recibirían las actualizaciones de un nuevo despliegue ni verían los efectos inmediatos de un **rollback**, ya que su navegador seguiría sirviendo el HTML obsoleto que apunta a hashes antiguos. Por esta razón, el HTML debe servirse con `Cache-Control: no-cache, must-revalidate` (o `max-age=0`), forzando al navegador a verificar siempre contra el servidor si hay una versión nueva antes de usar la copia local.

---

### Pregunta 7: Si la app necesitara SSR (Server-Side Rendering), ¿qué cambiaría en la arquitectura, en la CSP y en el costo?
1. **Cambio en la Arquitectura:**
   - Dejaría de ser un export estático servido directamente desde una CDN de almacenamiento plano.
   - Requeriría un servidor de aplicaciones en tiempo de ejecución (Node.js / contenedor) capaz de procesar requests entrantes y renderizar HTML dinámicamente.
   - En el ecosistema de Firebase, esto obligaría a migrar de **Firebase Hosting tradicional** a **Firebase App Hosting** o **Cloud Run / Cloud Functions**.
2. **Cambio en la CSP (Content-Security-Policy):**
   - **Mejora drástica de seguridad:** Al existir un servidor que procesa cada solicitud HTTP entrante, es posible generar un valor aleatorio criptográfico único por cada petición (**nonce**).
   - El servidor puede inyectar este nonce tanto en el encabezado CSP (`script-src 'self' 'nonce-rAnd0m123'`) como en las etiquetas `<script nonce="rAnd0m123">` de Next.js, permitiendo **eliminar por completo `'unsafe-inline'`** sin necesidad de precalcular hashes de contenido estático.
3. **Cambio en el Costo:**
   - El Plan **Spark** (gratuito) de Firebase no admite Cloud Run para App Hosting con tráfico de servidor dinámico.
   - Se requeriría migrar obligatoriamente al **Plan Blaze** (pago por consumo con tarjeta de crédito asociada) para sufragar el tiempo de cómputo de CPU/memoria de las instancias de Cloud Run, el tráfico de red de salida y las operaciones del contenedor.
