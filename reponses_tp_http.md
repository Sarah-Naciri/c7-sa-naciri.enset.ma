# Le Protocole HTTP - Solutions & Corrigés des Travaux Pratiques
**Réalisés par Sarah Naciri**

## TP 1 : Exploration avec les DevTools

- **Code de statut :** la requête GET retourne `200 OK`, indiquant que la requête a été traitée avec succès.
- **Headers envoyés :** le navigateur envoie automatiquement les en-têtes `Accept`, `User-Agent`, `Host` et `Cache-Control` pour décrire la requête et le client.
- **Content-Type :** la réponse du serveur est de type `application/json`, signifiant que le corps est au format JSON.
- **Erreur 405 :** cette erreur survient lorsqu'une méthode `GET` est utilisée sur un endpoint qui n'accepte que la méthode `POST`. Le serveur rejette la méthode non autorisée.

| URL | Méthode | Code | Content-Type |
| --- | --- | --- | --- |
| httpbin.org/get | GET | 200 OK | `application/json` |
| httpbin.org/post | POST | 200 OK | `application/json` |
| httpbin.org/status/201 | GET | 201 Created | `text/html` |
| httpbin.org/status/404 | GET | 404 Not Found | `text/html` |
| httpbin.org/status/500 | GET | 500 Server Error | `text/html` |
| httpbin.org/redirect/3 | GET | 302 Found | `text/html` |

## TP 2 : Maîtrise de cURL

- **`-i` vs `-v` :** L'option `-i` affiche les headers de réponse suivis du body, tandis que `-v` affiche le débogage complet de la communication (headers envoyés, reçus, négociation TLS, etc.).
- **POST Form vs JSON :** Un envoi en formulaire utilise le format `clé=valeur` (encodé URL), alors qu'un envoi JSON utilise un format structuré avec le header `Content-Type: application/json`.
- **Rôle de `-H` :** Permet d'ajouter des headers HTTP personnalisés à la requête (ex : `Authorization`, `Content-Type`, `X-Custom-Header`).
- **Rôle de `-L` :** Ordonne à cURL de suivre automatiquement les redirections HTTP (codes 3xx) jusqu'à atteindre la destination finale.
- **`-o` vs `-O` :** `-o` enregistre la réponse sous un nom de fichier personnalisé, tandis que `-O` conserve le nom d'origine du fichier distant.

Commande cURL complète (POST avec JSON, header personnalisé et affichage des headers de réponse) :

```bash
curl -i -X POST \
  -H "Content-Type: application/json" \
  -H "X-Custom-Header: MonHeader" \
  -d '{"action": "test", "value": 42}' \
  https://httpbin.org/post
```

## TP 3 : API REST avec JavaScript

- **`.then()` vs `async/await` :** Les deux gèrent des Promesses, mais `async/await` rend le code plus lisible et séquentiel, tandis que `.then()` enchaîne les callbacks.
- **Rôle de `POST` :** Créer une nouvelle ressource sur le serveur.
- **Rôle de `PUT` :** Modifier ou remplacer entièrement une ressource existante.
- **Rôle de `DELETE` :** Supprimer une ressource identifiée sur le serveur.
- **Principe de `fetchWithRetry` :** La fonction tente la requête et, en cas d'erreur de type 5xx (erreur serveur), elle réessaie automatiquement après un délai d'une seconde, jusqu'au nombre maximum de tentatives défini.

Implémentation de `fetchWithRetry` avec gestion des erreurs 5xx et délai entre tentatives :

```javascript
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  for (let i = 0; i <= maxRetries; i++) {
    try {
      const response = await fetch(url, options);

      // Si le code est une erreur serveur (500-599), on déclenche l'erreur
      if (response.status >= 500 && response.status < 600) {
        throw new Error(`Erreur HTTP : ${response.status}`);
      }

      return await response.json();

    } catch (error) {
      if (i === maxRetries) throw error; // Plus de tentatives possibles

      console.warn(`Tentative ${i + 1} échouée, nouvel essai dans 1s...`);

      // Pause de 1 seconde (1000 ms)
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

## TP 4 : Analyse des Headers de Sécurité

- **Rôle des headers de sécurité :** ils protègent les utilisateurs et les serveurs contre les attaques courantes comme le XSS (Cross-Site Scripting), le clickjacking et les attaques réseau de type man-in-the-middle.
- **Exemples essentiels :** `HSTS` (force HTTPS), `Content-Security-Policy` (restreint les sources de contenu), `X-Frame-Options` (empêche l'intégration dans des iframes malveillantes).

| Site | HSTS | X-Frame | CSP | Note estimée |
| --- | --- | --- | --- | --- |
| github.com | Oui (max-age=31536000) | DENY | Strict (default-src 'none') | A+ |
| google.com | Oui (max-age=31536000) | SAMEORIGIN | Oui (object-src 'none') | A |
| httpbin.org | Non | Absent | Absent | F |

## TP 5 : Cache HTTP

- **Rôle de `Cache-Control` :** définit la politique et la durée de mise en cache d'une ressource, permettant au navigateur ou aux proxies de stocker la réponse et d'éviter des requêtes réseau redondantes.
- **Rôle de `ETag` :** un identifiant unique de version d'une ressource. Il permet au client de vérifier si la ressource a changé depuis la dernière requête sans la re-télécharger entièrement.
- **Code `304 Not Modified` :** indique que la ressource demandée n'a pas changé depuis la dernière requête. Le navigateur peut donc utiliser directement la version en cache, évitant tout transfert de données.

Configuration des headers de cache recommandés (Nginx / Node.js) :
- **Fichier HTML :** `Cache-Control: no-cache` — Force le navigateur à toujours valider avec le serveur (ex: via ETag) avant d'afficher la page.
- **Fichier CSS / JS :** `Cache-Control: public, max-age=31536000, immutable` — Mis en cache pour 1 an. La bonne pratique est d'ajouter un hash au nom du fichier.
- **Image :** `Cache-Control: public, max-age=86400` — Mis en cache pour 1 jour.

## Questions Théoriques

1. **`no-cache` vs `no-store` :**
   `no-cache` autorise le stockage en cache mais force la revalidation auprès du serveur avant chaque utilisation. `no-store` interdit tout stockage de la réponse, même temporaire — recommandé pour les données sensibles.
2. **Pourquoi `POST` n'est pas idempotent :**
   Chaque appel `POST` modifie l'état du serveur en créant une nouvelle ressource. Répéter la même requête produira donc des effets cumulatifs, contrairement à `GET` ou `PUT` qui sont idempotents.
3. **Code `301` :**
   Il s'agit d'une *redirection permanente*. Le navigateur suit automatiquement le header `Location` et met en cache la redirection pour les futures visites.
4. **Header `Origin` :**
   Indique la source (protocole + domaine + port) de la requête. Le serveur l'utilise pour appliquer les politiques CORS et décider d'autoriser ou refuser la requête cross-origine.
5. **HttpOnly sur les cookies :**
   L'attribut `HttpOnly` empêche tout accès aux cookies via JavaScript (`document.cookie`), protégeant ainsi les sessions contre le vol lors d'attaques XSS.
