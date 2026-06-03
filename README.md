netlify/functions/analyse.js
// =============================================================
//  FONCTION NETLIFY — Analyse IA securisee
//  Le site appelle cette fonction, qui detient la cle API secrete.
//  La cle n'est JAMAIS exposee cote navigateur.
// =============================================================

export default async (request) => {
  const cors = {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
  };

  if (request.method === "OPTIONS") {
    return new Response("", { status: 204, headers: cors });
  }
  if (request.method !== "POST") {
    return new Response(JSON.stringify({ error: "Methode non autorisee" }), {
      status: 405,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }

  const API_KEY = process.env.ANTHROPIC_API_KEY;
  if (!API_KEY) {
    return new Response(
      JSON.stringify({ error: "Cle API manquante cote serveur (ANTHROPIC_API_KEY)" }),
      { status: 500, headers: { ...cors, "Content-Type": "application/json" } }
    );
  }

  let detail = "";
  try {
    const body = await request.json();
    detail = (body && body.detail) ? String(body.detail) : "";
  } catch (e) {
    return new Response(JSON.stringify({ error: "Corps de requete invalide" }), {
      status: 400,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }

  const PROMPT_ANALYSE =
`Tu es un expert en recrutement pour un poste d'animateur de lives TikTok (jeune public).
Tu evalues la "jugeote" d'un candidat : debrouillardise, bon sens, autonomie, initiative et fiabilite dans la vie de tous les jours (pas seulement au travail).

Le candidat a aussi indique s'il dispose des PREREQUIS suivants : un compte TikTok, un PC/ordinateur portable, une connexion internet stable, un endroit calme pour live, et 1 a 2h par jour de disponibilite. Tiens-en compte dans le score : un candidat a qui il manque des prerequis essentiels (PC, connexion, disponibilite) doit voir son score et sa recommandation revus a la baisse.

Evalue la QUALITE DU RAISONNEMENT, pas une grille rigide. Les questions n'ont pas de bonne reponse unique.

Voici les reponses du candidat :
===DEBUT===
${detail}
===FIN===

Reponds UNIQUEMENT avec un objet JSON valide, sans texte autour, sans backticks markdown, exactement dans ce format :
{
  "score": 7,
  "synthese": "resume du profil en 2-3 phrases",
  "points_forts": ["point 1", "point 2", "point 3"],
  "points_faibles": ["point 1", "point 2"],
  "prerequis": "resume en 1 phrase de ce que le candidat possede ou pas",
  "recommandation": "RECRUTER ou A APPROFONDIR ou NE PAS RECRUTER",
  "justification": "pourquoi cette recommandation, 1-2 phrases"
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
    text = text.split("```json").join("").split("```").join("").trim();

    const analysis = JSON.parse(text);

    return new Response(JSON.stringify(analysis), {
      status: 200,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  } catch (e) {
    return new Response(JSON.stringify({ error: "Echec de l'analyse", message: String(e) }), {
      status: 500,
      headers: { ...cors, "Content-Type": "application/json" },
    });
  }
};
