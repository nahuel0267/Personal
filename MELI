// api/trends.js - Vercel Serverless Function
// Llama a la API real de Mercado Libre y devuelve tendencias + categorías

const CLIENT_ID = process.env.MELI_CLIENT_ID;
const CLIENT_SECRET = process.env.MELI_CLIENT_SECRET;
const SITE_ID = "MLA"; // Argentina

// Cache en memoria (dura mientras la función esté "caliente" en Vercel)
// Para producción seria, usar Vercel KV o Redis
let tokenCache = { token: null, expiresAt: 0 };

async function getAccessToken() {
  const now = Date.now();
  // Si el token sigue vigente (con 5 min de margen), lo reutilizamos
  if (tokenCache.token && now < tokenCache.expiresAt - 300000) {
    return tokenCache.token;
  }

  const res = await fetch("https://api.mercadolibre.com/oauth/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "client_credentials",
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
    }),
  });

  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Error obteniendo token MELI: ${res.status} - ${err}`);
  }

  const data = await res.json();
  tokenCache = {
    token: data.access_token,
    expiresAt: now + data.expires_in * 1000, // expires_in viene en segundos
  };

  return tokenCache.token;
}

// Categorías principales de MLA con sus IDs reales
const CATEGORIAS = [
  { id: "MLA1051", nombre: "Celulares y Teléfonos" },
  { id: "MLA1000", nombre: "Electrónica" },
  { id: "MLA1246", nombre: "Belleza y Cuidado" },
  { id: "MLA1430", nombre: "Ropa y Calzado" },
  { id: "MLA1574", nombre: "Hogar y Jardín" },
  { id: "MLA1276", nombre: "Deportes" },
  { id: "MLA1132", nombre: "Computación" },
  { id: "MLA5726", nombre: "Juguetes" },
];

async function getTrends(token) {
  // 1. Tendencias generales (top 50 de toda la plataforma)
  const generalRes = await fetch(`https://api.mercadolibre.com/trends/${SITE_ID}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const generalTrends = generalRes.ok ? await generalRes.json() : [];

  // 2. Tendencias por categoría (en paralelo para no esperar una por una)
  const catPromises = CATEGORIAS.map(async (cat) => {
    try {
      const res = await fetch(
        `https://api.mercadolibre.com/trends/${SITE_ID}/${cat.id}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      if (!res.ok) return { ...cat, trends: [] };
      const trends = await res.json();
      return { ...cat, trends: trends.slice(0, 5) }; // Top 5 por categoría
    } catch {
      return { ...cat, trends: [] };
    }
  });

  const categoriasTrends = await Promise.all(catPromises);

  // Estructura de respuesta:
  // generalTrends[0-9]   = mayor crecimiento (últimas 2 semanas)
  // generalTrends[10-29] = más buscados (semana)
  // generalTrends[30-49] = más populares (semana)
  return {
    general: {
      mayor_crecimiento: generalTrends.slice(0, 10),
      mas_buscados: generalTrends.slice(10, 30),
      mas_populares: generalTrends.slice(30, 50),
    },
    por_categoria: categoriasTrends,
    metadata: {
      timestamp: new Date().toISOString(),
      site: SITE_ID,
      fuente: "Mercado Libre API oficial",
    },
  };
}

export default async function handler(req, res) {
  // CORS - permite llamadas desde cualquier origen (el frontend en GitHub Pages, etc.)
  res.setHeader("Access-Control-Allow-Origin", "*");
  res.setHeader("Access-Control-Allow-Methods", "GET, OPTIONS");
  res.setHeader("Cache-Control", "s-maxage=3600, stale-while-revalidate=7200");
  // Cache de 1 hora en el CDN de Vercel — la API de MELI se actualiza semanalmente
  // así que no tiene sentido llamarla más seguido

  if (req.method === "OPTIONS") {
    return res.status(200).end();
  }

  if (!CLIENT_ID || !CLIENT_SECRET) {
    return res.status(500).json({
      error: "Variables de entorno MELI_CLIENT_ID y MELI_CLIENT_SECRET no configuradas",
    });
  }

  try {
    const token = await getAccessToken();
    const data = await getTrends(token);
    return res.status(200).json(data);
  } catch (err) {
    console.error("Error en /api/trends:", err.message);
    return res.status(500).json({ error: err.message });
  }
}
