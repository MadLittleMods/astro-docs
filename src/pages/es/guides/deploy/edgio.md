---
title: Despliega tu sitio Astro en Edgio
description: Cómo desplegar tu sitio Astro en la web usando Edgio.
layout: ~/layouts/DeployGuideLayout.astro
i18nReady: true
---

Puedes desplegar tu proyecto de Astro en [Edgio](https://www.edg.io/), una plataforma de CDN y edge para desplegar, proteger y acelerar sitios web y APIs.

:::tip
¡Échale un vistazo a [la guía de Astro en la documentación de Edgio](https://docs.edg.io/guides/astro)!
:::

## ¿Cómo desplegar?

1. Instala [la CLI de Edgio](https://docs.edg.io/guides/cli) globalmente desde la Terminal, si aún no lo has hecho.

    ```bash
    npm install -g @edgio/cli
    ```

2. Agrega Edgio a tu sitio Astro.

    ```bash
    edgio init
    ```

3. (Opcional) Habilita el Server Side Rendering.

    Después de configurar [Server Side Rendering con Astro](/es/guides/server-side-rendering/), especifica la ruta del archivo del servidor en `edgio.config.js` como se muestra a continuación:

    ```js title="edgio.config.js" ins={1,4-8}
    import { join } from 'path'

    module.exports = {
      astro: {
        // La ruta del servidor independiente que ejecuta SSR de Astro.
        // Las dependencias para este archivo se empaquetan automáticamente.
        appPath: join(process.cwd(), 'dist', 'server', 'entry.mjs'),
      },
    };
    ```

4. Despliega a Edgio.

    ```bash
    edgio deploy
    ```
