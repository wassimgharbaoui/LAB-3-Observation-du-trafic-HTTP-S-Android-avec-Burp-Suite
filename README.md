# LAB 3 — Interception du trafic HTTP/HTTPS Android avec Burp Suite

## Présentation

Ce laboratoire porte sur l'observation et l'analyse des communications réseau générées par un émulateur Android. Le principe repose sur le positionnement de **Burp Suite Community Edition** en tant que **proxy man-in-the-middle** entre l'émulateur et internet : tout le trafic HTTP et HTTPS transite par Burp avant d'atteindre sa destination, ce qui permet de le visualiser, l'inspecter et éventuellement le modifier.

Ce type d'analyse est fondamental en sécurité mobile — elle permet de détecter des transmissions de données non chiffrées, des tokens exposés dans les URLs, des cookies mal configurés ou encore des endpoints non documentés.

| Paramètre | Valeur |
|---|---|
| Outil principal | Burp Suite Community Edition |
| Environnement | Émulateur Android (Mobexler / VirtualBox) |
| Auteur | Wassim Gharbaoui |
| Périmètre | Sites et services autorisés uniquement |

---

## Étape 1 — Démarrage de Burp Suite et configuration initiale

Au lancement, Burp Suite propose deux modes de projet : un projet sauvegardé sur disque ou un **projet temporaire en mémoire**. Pour un audit de laboratoire, le mode temporaire est suffisant — aucune donnée n'est persistée sur le disque, ce qui convient à un environnement de test.

Une fois Burp ouvert, la première vérification à faire est l'état de l'interception. Il est recommandé de **laisser l'interception désactivée** (`Intercept is off`) à ce stade : si elle est active avant que le proxy soit configuré sur l'émulateur, toutes les requêtes entrantes seront mises en attente et bloqueront la navigation sans raison.


<img width="777" height="350" alt="Screenshot 2026-03-18 015422" src="https://github.com/user-attachments/assets/852c0363-c40e-4269-8509-0dc61177c05f" />

<img width="781" height="358" alt="Screenshot 2026-03-18 015429" src="https://github.com/user-attachments/assets/a2de1887-60e6-4fbf-9262-49a54470be2a" />


---

## Étape 2 — Configuration du listener proxy

Le **listener** est le composant de Burp Suite qui écoute les connexions entrantes. Par défaut, il est configuré sur `127.0.0.1:8080` — ce qui signifie qu'il n'accepte que des connexions locales (loopback). Or, l'émulateur Android est une machine virtuelle distincte : il ne peut pas atteindre le loopback de la machine hôte.

Il faut donc modifier cette configuration pour que Burp écoute sur **toutes les interfaces réseau** (`All interfaces` → `0.0.0.0:8080`). Cela rend le proxy accessible depuis n'importe quelle machine sur le réseau local, y compris l'émulateur.

> ⚠️ En environnement de production ou sur un réseau partagé, écouter sur toutes les interfaces représente un risque. Dans un lab isolé, c'est acceptable.

Chemin dans Burp : **Proxy → Settings → Proxy listeners → Edit → Binding tab → All interfaces**


<img width="778" height="349" alt="Screenshot 2026-03-18 015450" src="https://github.com/user-attachments/assets/deef7799-c42d-4cd2-8b84-1dd5bfdf9d7c" />

<img width="779" height="406" alt="Screenshot 2026-03-18 015459" src="https://github.com/user-attachments/assets/dc7d39f8-0c22-4769-831e-16f194b689e7" />

<img width="780" height="283" alt="Screenshot 2026-03-18 015507" src="https://github.com/user-attachments/assets/4307e72a-c9ec-4366-b155-c2c68531b330" />


---

## Étape 3 — Identification de l'adresse IP de la machine hôte

Pour que l'émulateur sache où envoyer son trafic, il a besoin de l'adresse IP de la machine hôte sur le réseau local. Cette adresse est récupérée avec la commande suivante :

```bash
# Linux / macOS
ip a

# Windows
ipconfig
```

Dans ce laboratoire, l'interface réseau utilisée est `enp0s17` avec l'adresse **192.168.1.196**. C'est cette valeur qui sera renseignée dans la configuration proxy de l'émulateur.

> Il faut s'assurer d'utiliser l'adresse IP de l'interface connectée au même réseau que l'émulateur, et non le loopback (`127.0.0.1`).

<img width="773" height="561" alt="Screenshot 2026-03-18 015515" src="https://github.com/user-attachments/assets/8a036f4b-86b1-4d71-b681-82c6df6e8bf9" />

---

## Étape 4 — Paramétrage du proxy sur l'émulateur Android

L'émulateur Android doit maintenant être configuré pour faire transiter tout son trafic HTTP via Burp Suite. Cette configuration se fait au niveau du réseau Wi-Fi.

**Chemin :** Paramètres → Wi-Fi → réseau connecté → Options avancées → Proxy → Manuel

Renseigner :
- **Proxy hostname** : `192.168.1.196` (IP de la machine hôte identifiée à l'étape précédente)
- **Proxy port** : `8080`

Dès que cette configuration est validée, tout le trafic HTTP de l'émulateur sera redirigé vers Burp Suite. Le trafic HTTPS sera également redirigé, mais restera illisible tant que le certificat CA de Burp n'est pas installé (traité à l'étape 7).


<img width="350" height="645" alt="Screenshot 2026-03-18 015532" src="https://github.com/user-attachments/assets/fcabc52e-2e8d-4215-b05e-5930a5549abd" />

---

## Étape 5 — Vérification de la capture du trafic

Pour confirmer que le proxy fonctionne, on navigue sur quelques sites depuis le navigateur de l'émulateur (par exemple `http://example.com`, `http://speedtest.net`). Il est conseillé de commencer par des URLs en **HTTP pur** pour valider la chaîne sans la complexité du chiffrement TLS.

Dans Burp Suite, l'onglet **Proxy → HTTP history** liste toutes les requêtes capturées. Chaque ligne affiche les informations essentielles :

- **Host** — domaine cible de la requête
- **Method** — GET, POST, PUT…
- **URL** — chemin et paramètres de la requête
- **Status** — code de réponse HTTP (200, 301, 404…)
- **Length** — taille de la réponse en octets
- **MIME type** — nature du contenu retourné (text/html, application/json…)

La présence de ces entrées dans l'historique confirme que le proxy intercepte bien le trafic.

<img width="387" height="651" alt="Screenshot 2026-03-18 015538" src="https://github.com/user-attachments/assets/a1516285-9f5d-48ac-a545-13d17b8e7ff4" />


<img width="774" height="272" alt="Screenshot 2026-03-18 015550" src="https://github.com/user-attachments/assets/aa2989f6-1ee1-434d-9211-553e154477a8" />

---

## Étape 6 — Analyse détaillée d'une requête

En cliquant sur une requête dans l'historique, Burp propose deux vues complémentaires :

**Vue Raw** — affiche la requête HTTP brute telle qu'elle a été transmise :
```
GET /index.html HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0 ...
Accept: text/html
Cookie: session=abc123
```

**Vue Inspector** — présente les mêmes informations de manière structurée et navigable : paramètres de requête, headers de requête, headers de réponse, cookies.

Points d'attention lors de l'analyse :
- Des **tokens ou identifiants dans l'URL** (ex. `?token=xyz`) — ils apparaissent dans les logs serveurs et l'historique du navigateur, ce qui constitue une fuite potentielle.
- Des **cookies sans attributs de sécurité** (`Secure`, `HttpOnly`, `SameSite`) — vulnérables au vol ou à la manipulation.
- Des **données personnelles transmises en clair** dans le corps de la requête (formulaires non chiffrés).

<img width="779" height="303" alt="Screenshot 2026-03-18 015559" src="https://github.com/user-attachments/assets/f7a0ad0d-eedc-4c26-9ac2-45d481822749" />

---

## Étape 7 — Interception active et modification de requêtes

L'interception active est la fonctionnalité la plus puissante de Burp Suite en phase de test : elle permet de **bloquer une requête avant qu'elle soit envoyée** et de la modifier à la volée.

Pour l'activer : **Proxy → Intercept → Intercept is on**

Une fois activée, chaque requête émise par l'émulateur est mise en pause dans Burp jusqu'à une action manuelle :
- **Forward** — la requête est transmise au serveur telle quelle (ou modifiée).
- **Drop** — la requête est abandonnée, le serveur ne la reçoit pas.
- **Action** — donne accès à des options avancées (envoyer vers le Repeater, le Scanner, etc.)

> Penser à **désactiver l'interception** après la démonstration (`Intercept is off`). Laisser l'interception active bloque tout le trafic de l'émulateur et peut sembler être un dysfonctionnement.

<img width="776" height="295" alt="Screenshot 2026-03-18 015605" src="https://github.com/user-attachments/assets/ecc483d9-de92-4806-965b-f3fb9c167b18" />


<img width="776" height="293" alt="Screenshot 2026-03-18 015615" src="https://github.com/user-attachments/assets/4af6d093-d7e7-4439-9abe-91d557348d31" />


---

## Étape 8 — Déchiffrement HTTPS : installation du certificat CA Burp

Par défaut, Burp Suite génère ses propres certificats TLS pour chaque domaine qu'il intercepte. Ces certificats sont signés par une **autorité de certification (CA) propriétaire à Burp**, inconnue du système Android. Résultat : Android refuse la connexion et affiche une erreur de certificat.

Pour que le trafic HTTPS soit déchiffrable, il faut **importer la CA de Burp dans le magasin de confiance d'Android**. Une fois installée, Android considère les certificats générés par Burp comme légitimes et laisse passer les connexions — ce qui permet à Burp de déchiffrer le contenu HTTPS, de l'inspecter, puis de le re-chiffrer avant de l'envoyer au serveur réel.

**Comment exporter le certificat depuis Burp :**
Burp Suite → Proxy → Settings → Import/export CA certificate → Export → Certificate in DER format → sauvegarder sous `cacert.der`

**Installation sur l'émulateur Android :**
Paramètres → Sécurité → Chiffrement et identifiants → Installer un certificat → Certificat CA → sélectionner le fichier exporté

> ⚠️ **Avertissement important** : Ce certificat ne doit jamais être installé sur un appareil personnel ou de production. Il donne à Burp Suite la capacité de déchiffrer **tout** le trafic HTTPS de l'appareil. À utiliser exclusivement en environnement de laboratoire isolé, et à supprimer impérativement en fin de session.


<img width="377" height="721" alt="Screenshot 2026-03-18 015626" src="https://github.com/user-attachments/assets/4d26c230-9e77-4bfa-96f1-caba45117c66" />

<img width="388" height="609" alt="Screenshot 2026-03-18 015648" src="https://github.com/user-attachments/assets/a48e7751-c9e0-44db-b9e0-04bdd240d6ac" />

<img width="375" height="404" alt="Screenshot 2026-03-18 015653" src="https://github.com/user-attachments/assets/f9d1f3be-0453-43f7-bb6d-c83c39f115c5" />



## Bonnes pratiques de sécurité réseau mobile

Ce laboratoire met en évidence plusieurs vulnérabilités courantes dans les applications mobiles. Voici les pratiques défensives correspondantes :

**Côté transmission des données**
- Utiliser **HTTPS pour toutes les communications**, sans exception — le HTTP en clair est interceptable sans même avoir besoin d'un certificat.
- Ne jamais transmettre des tokens ou identifiants **dans l'URL** (ils sont loggés par les serveurs web et visibles dans l'historique navigateur). Utiliser les **headers d'autorisation** à la place (`Authorization: Bearer ...`).

**Côté cookies**
- Toujours définir les attributs `Secure` (cookie transmis uniquement en HTTPS), `HttpOnly` (inaccessible au JavaScript) et `SameSite=Strict` (protection contre le CSRF).

**Côté application**
- Mettre en place le **Certificate Pinning** : l'application ne fait confiance qu'à un certificat ou une clé publique spécifique, ce qui bloque les proxies comme Burp même si leur CA est installée. C'est la contre-mesure directe à ce que nous avons réalisé ici.
- Limiter les données transmises au strict nécessaire — ne jamais envoyer des champs non utilisés par le serveur.

---

## Nettoyage post-laboratoire

Une fois le laboratoire terminé, l'environnement doit être remis dans son état initial pour ne laisser aucune trace et ne pas compromettre de futures sessions :

1. Sur l'émulateur : **Paramètres Wi-Fi → Options avancées → Proxy → None**
2. Sur l'émulateur : **Paramètres → Sécurité → Certificats CA → supprimer le certificat Burp**
3. Fermer Burp Suite — le projet temporaire est automatiquement effacé de la mémoire.
4. Conserver uniquement les captures d'écran et notes nécessaires au rapport d'audit.
