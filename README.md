netlify/functions/analyse.js
// =============================================================
//  FONCTION NETLIFY — Analyse IA sécurisée
//  Le site appelle cette fonction, qui détient la clé API secrète.
//  La clé n'est JAMAIS exposée côté navigateur.
// =============================================================

export default async (request) => {
  // CORS : autorise ton site à appeler la fonction
  const cors = {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
  };

  // Pré-vol CORS
  if (request.method === "OPTIONS") {
    return new Response("", { status: 204, headers: cors });
  }
  if (request.method !== "POST") {
    return new Response(JSON.stringify({ error: "Méthode non autorisée" }), {
      status: 405,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }

  // La clé est lue depuis les variables d'environnement Netlify (jamais en dur)
  const API_KEY = process.env.ANTHROPIC_API_KEY;
  if (!API_KEY) {
    return new Response(
      JSON.stringify({ error: "Clé API manquante côté serveur (ANTHROPIC_API_KEY)" }),
      { status: 500, headers: { ...cors, "Content-Type": "application/json" } }
    );
  }

  let detail = "";
  try {
    const body = await request.json();
    detail = (body && body.detail) ? String(body.detail) : "";
  } catch (e) {
    return new Response(JSON.stringify({ error: "Corps de requête invalide" }), {
      status: 400,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }

  // ---- Le prompt d'analyse ----
  const PROMPT_ANALYSE =
`Tu es un expert en recrutement pour un poste d'animateur de lives TikTok (jeune public).
Tu évalues la "jugeote" d'un candidat : débrouillardise, bon sens, autonomie, initiative et fiabilité dans la vie de tous les jours (pas seulement au travail).

Le candidat a aussi indiqué s'il dispose des PRÉREQUIS suivants : un compte TikTok, un PC/ordinateur portable, une connexion internet stable, un endroit calme pour live, et 1 à 2h par jour de disponibilité. Tiens-en compte dans le score : un candidat à qui il manque des prérequis essentiels (PC, connexion, disponibilité) doit voir son score et sa recommandation revus à la baisse.

Évalue la QUALITÉ DU RAISONNEMENT, pas une grille rigide. Les questions n'ont pas de bonne réponse unique.

Voici les réponses du candidat :
---
${detail}
---

Réponds UNIQUEMENT avec un objet JSON valide, sans texte autour, sans backticks markdown, exactement dans ce format :
{
  "score": <entier de 0 à 10>,
  "synthese": "<résumé du profil en 2-3 phrases>",
  "points_forts": ["<point 1>", "<point 2>", "<point 3>"],
  "points_faibles": ["<point 1>", "<point 2>"],
  "prerequis": "<résumé en 1 phrase de ce que le candidat possède ou pas>",
  "recommandation": "<RECRUTER | À APPROFONDIR | NE PAS RECRUTER>",
  "justification": "<pourquoi cette recommandation, 1-2 phrases>"
}`;

  try {
    const aiResp = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-api-key": API_KEY,
        "anthropic-version": "2023-06-01",
      },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 1000,
        messages: [{ role: "user", content: PROMPT_ANALYSE }],
      }),
    });

    const data = await aiResp.json();

    if (!aiResp.ok) {
      return new Response(JSON.stringify({ error: "Erreur API Anthropic", detail: data }), {
        status: 502,
        headers: { ...cors, "Content-Type": "application/json" },
      });
    }

    let text = (data.content || [])
      .filter((b) => b.type === "text")
      .map((b) => b.text)
      .join("");
    text = text.replace(/```json/gi, "").replace(/```/g, "").trim();

    const analysis = JSON.parse(text);

    return new Response(JSON.stringify(analysis), {
      status: 200,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  } catch (e) {
    return new Response(JSON.stringify({ error: "Échec de l'analyse", message: String(e) }), {
      status: 500,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }
};
