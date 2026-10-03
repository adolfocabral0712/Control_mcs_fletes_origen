const ROUTES = {
  "/api/remitos": "REMITOS_JSON_URL",
  "/api/cotizaciones": "COTIZACIONES_JSON_URL",
};

function jsonError(message, status) {
  return Response.json(
    { error: message },
    {
      status,
      headers: {
        "Cache-Control": "no-store",
        "X-Content-Type-Options": "nosniff",
      },
    }
  );
}

export default {
  async fetch(request, env) {
    const pathname = new URL(request.url).pathname;

    if (!pathname.startsWith("/api/")) {
      return env.ASSETS.fetch(request);
    }

    const secretName = ROUTES[pathname];

    if (!secretName) {
      return jsonError("API no encontrada.", 404);
    }

    if (request.method !== "GET" && request.method !== "HEAD") {
      return new Response("Método no permitido.", {
        status: 405,
        headers: { Allow: "GET, HEAD" },
      });
    }

    const secret = env[secretName];

    if (!secret) {
      return jsonError(
        `Falta configurar el secreto ${secretName}.`,
        503
      );
    }

    let source;

    try {
      source = new URL(secret.trim());

      if (
        source.protocol !== "https:" ||
        !["dl.dropbox.com", "www.dropbox.com"].includes(source.hostname)
      ) {
        return jsonError(
          `El secreto ${secretName} debe ser un enlace HTTPS de Dropbox.`,
          503
        );
      }

      source.hostname = "dl.dropbox.com";
      source.searchParams.set("dl", "1");
    } catch {
      return jsonError(
        `El secreto ${secretName} no contiene una URL válida.`,
        503
      );
    }

    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), 90000);

    try {
      const upstream = await fetch(source.toString(), {
        signal: controller.signal,
        redirect: "follow",
        headers: {
          Accept: "application/json, text/plain;q=0.9",
        },
        cf: {
          cacheTtl: 0,
          cacheEverything: false,
        },
      });

      if (!upstream.ok) {
        return jsonError(
          "No se pudo leer el archivo de datos.",
          502
        );
      }

      const data = await upstream.json();

      if (!Array.isArray(data) && !Array.isArray(data?.datos)) {
        return jsonError(
          "El archivo recibido no tiene el formato JSON esperado.",
          502
        );
      }

      return new Response(
        request.method === "HEAD" ? null : JSON.stringify(data),
        {
          headers: {
            "Content-Type": "application/json; charset=utf-8",
            "Cache-Control": "no-store",
            "X-Content-Type-Options": "nosniff",
          },
        }
      );
    } catch {
      return jsonError(
        "No se pudo descargar o interpretar el JSON. Intentá actualizar nuevamente.",
        502
      );
    } finally {
      clearTimeout(timer);
    }
  },
};
