# Documentación DataMart

Sitio de documentación del proyecto de analítica de ventas, construido con
Docusaurus. El contenido incluye arquitectura y decisiones técnicas, consultas
SQL, recomendación de negocio y documentación de la API de productos.

## Desarrollo local

Requiere Node.js 20 o superior.

```powershell
npm install
npm run start
```

Para generar y revisar la versión estática:

```powershell
npm run build
npm run serve
```

El sitio generado queda en `build/`.

## Firebase Hosting

Firebase Hosting aún no está configurado. Al inicializarlo, usa `build` como
directorio público y configura las reescrituras de SPA según la guía de
Docusaurus. Antes del despliegue, actualiza `url` en `docusaurus.config.js` con
la URL asignada al sitio Firebase; luego genera el sitio y despliega con la
herramienta Firebase CLI.

El README del repositorio del proyecto contiene el enlace provisional que se
debe actualizar con la URL final de Firebase.
