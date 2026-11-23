# Compte rendu : Observer HTTP, du navigateur à Wireshark

**Cours :** Architectures et applications web

**Date du TP :** 20 novembre 2026

Objectif : observer le même échange HTTP à deux niveaux, d'abord dans le navigateur (requêtes décodées), puis dans Wireshark (paquets réellement échangés sur le réseau).

---

## Environnement de test

| Élément | Valeur observée |
| --- | --- |
| Serveur web | Apache/2.4.65 (Win64), OpenSSL/3.5.1, PHP/8.3.36 |
| Navigateur | Brave 158 (base Chromium), sous Windows (`sec-ch-ua`, `User-Agent`) |
| Outil de capture | Wireshark, interface « Adapter for loopback traffic capture » (Npcap) |
| Dossier de test | `http://localhost/test_http/` |
| Pages de test | `formulaire.php` (GET et POST), `redirige.php` (redirection 302) |

---

## Partie 1 : Observer dans le navigateur

Onglet **Réseau** (F12) ouvert, avec *Désactiver le cache* et *Conserver les journaux* cochés.

### Étape 1 : Une requête simple

Chargement de `http://localhost/test_http/formulaire.php`.

| Élément à relever | Où le trouver | Valeur observée |
| --- | --- | --- |
| Méthode | En-têtes, ligne de requête | `GET` |
| URL complète | En-têtes | `http://localhost/test_http/formulaire.php` |
| Code de statut | En-têtes, ligne de statut | `200 OK` |
| Version de HTTP | En-têtes | `HTTP/1.1` |
| `Host` | En-têtes de requête | `localhost` |
| `User-Agent` | En-têtes de requête | `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/158.0.0.0 Safari/537.36` |
| `Content-Type` | En-têtes de réponse | `text/html; charset=UTF-8` |
| `Content-Length` | En-têtes de réponse | `385` (octets) |
| `Server` | En-têtes de réponse | `Apache/2.4.65 (Win64) OpenSSL/3.5.1 PHP/8.3.36` |
| Durée totale | Onglet Délais (*Timings*) | 40 ms (lecture de la frise chronologique) |

**Observations :**

- `Content-Length: 385` est non nul : la page a bien été générée en entier (pas de page blanche due à une erreur PHP).
- Les en-têtes `Server` et `X-Powered-By` révèlent les versions d'Apache, d'OpenSSL et de PHP. Acceptable en développement ; en production on les masque (`ServerTokens Prod`, `expose_php = Off`).
- Le `User-Agent` annonce « Chrome/158 » : il ne dit pas que le navigateur est Brave. C'est l'en-tête `sec-ch-ua` (Client Hints) qui le révèle (`"Brave";v="158"`).

### Étape 2 : Envoyer en GET

Saisie de `emmanuel` dans le premier formulaire, puis clic sur *Envoyer en GET*.

- La barre d'adresse devient `http://localhost/test_http/formulaire.php?nom=emmanuel` et la page affiche « Bonjour emmanuel ».
- L'onglet Réseau montre une nouvelle requête (statut 200, type `document`). La rubrique *Payload* liste le paramètre `nom` = `emmanuel` sous *Query string parameters*.
- **Conclusion :** en GET, les données voyagent dans l'URL (la *query string*) et arrivent côté PHP dans `$_GET['nom']`.

### Étape 3 : Envoyer en POST

Saisie du nom dans le second formulaire, puis clic sur *Envoyer en POST*.

- La méthode de la requête est `POST`, le statut `200 OK`, et l'URL ne change pas.
- `Content-Length` passe de 385 à **393** : la différence est de 8 octets, soit exactement les 8 caractères de « emmanuel » ajoutés dans le titre `Bonjour ...`.

### Étape 4 : Une erreur 404

Chargement de `http://localhost/test_http/absent.php`.

- Code de statut : **404 Not Found** (repéré par la pastille rouge dans la liste).
- La page affiche « Not Found », « The requested URL was not found on this server. »

### Étape 5 : Une redirection

Chargement de `http://localhost/test_http/redirige.php`.

- Deux requêtes apparaissent. La première, `redirige.php`, répond **302 Found** avec un corps vide (`Content-Length: 0`). La seconde est un `GET formulaire.php`, qui répond 200.
- L'en-tête `Location` de la première réponse vaut `formulaire.php` (valeur fixée par `header('Location: formulaire.php')` dans le code).

### Étape 6 : Le cache

- Observation faite sur `favicon.ico` : statut **`200 OK (from disk cache)`**. Le navigateur a réutilisé sa copie locale sans contacter le serveur.
- La réponse stockée porte `Etag: "78ae-51affc7a4c400"` et `Last-Modified: Thu, 16 Jul 2015 15:32:32 GMT`. Ce sont ces validateurs qui permettent, une fois la copie périmée, de demander au serveur « a-t-elle changé ? » (`If-None-Match` / `If-Modified-Since`) et de recevoir un `304 Not Modified` sans le corps.
- Après chaque chargement de `formulaire.php`, la liste montre un `favicon.ico` : le navigateur le redemande à chaque fois, mais le récupère dans son cache.

### Étape 7 : Un vrai site

Chargement de la page d'accueil de Wikipédia en français (`fr.wikipedia.org/wiki/Wikipédia:Accueil_principal`). 

| Mesure relevée | Valeur observée |
| --- | --- |
| Nombre de requêtes | 36 |
| Volume transféré | 11,5 kB |
| Taille totale des ressources | 2,3 MB |
| Fin de toutes les requêtes (*Finish*) | 1,29 s |
| Structure de la page prête (*DOMContentLoaded*) | 576 ms |
| Chargement complet (*Load*) | 952 ms |
| Versions de HTTP | à compléter (colonne *Protocole* non affichée sur la capture) |

**Observations :**

- **Une page réelle, c'est beaucoup de requêtes.** Notre page locale `formulaire.php` n'en demandait qu'une (640 octets). Ici, l'affichage d'une seule page déclenche 36 requêtes de types variés : document HTML, feuilles de style, scripts, logos (`png`, `svg+xml`) et images (`webp`).
- **Écart entre 2,3 MB et 11,5 kB.** La page pèse 2,3 MB de ressources, mais seuls 11,5 kB ont réellement circulé sur le réseau. L'écart vient essentiellement du cache : la plupart des ressources n'ont pas été retéléchargées.
- **Le cache en action.** Trois codes de statut apparaissent dans la liste :
    - **304 Not Modified** pour le document principal (166 ms), le script de démarrage (205 ms) et une image (171 ms) : le navigateur a demandé au serveur si sa copie était à jour, et le serveur a répondu que oui, sans renvoyer la ressource.
    - **`(memory cache)`** pour les logos et la plupart des images : 0 ms, aucune requête n'est partie.
    - **`(disk cache)`** pour deux feuilles de style : 117 ms et 127 ms.

---

## Partie 2 : Installation de Wireshark

Wireshark est installé avec Npcap et l'option *Support loopback traffic*, ce qui permet de capturer les échanges avec `localhost`. La capture sur l'interface « Adapter for loopback traffic capture » fonctionne (Partie 3).

---

## Partie 3 : Capturer et analyser avec Wireshark

### Étapes 1 à 3 : Interface, capture et filtre

- Interface choisie : **Adapter for loopback traffic capture** (les échanges avec `localhost` ne passent pas par le Wi-Fi).
- Action provoquée : rechargement de `http://localhost/test_http/formulaire.php`, puis arrêt de la capture.
- Filtre d'affichage utilisé : `tcp.port == 80` (barre verte, filtre valide). Wireshark affiche **14 paquets sur 90 254 capturés** : c'est tout l'intérêt du filtre.

### Étape 4 : Lire les couches

| Couche (cours, ch. 1.4) | Ligne dans Wireshark | Information relevée |
| --- | --- | --- |
| Accès réseau | `Null/Loopback` | Trame de la boucle locale : en-tête de 4 octets, pas d'adresse MAC |
| Internet | `Internet Protocol Version 6` | Source `::1`, destination `::1` (adresse de bouclage IPv6) |
| Transport | `Transmission Control Protocol` | Port source **6716** (port éphémère du navigateur), port destination **80** |
| Application | `Hypertext Transfer Protocol` | `GET /test_http/formulaire.php HTTP/1.1`, avec les en-têtes `Host: localhost`, `Connection: keep-alive`, `sec-ch-ua`, `User-Agent`, `Accept`, `Sec-Fetch-*`... |

**Contrôle par les tailles :** la requête fait 722 octets sur le réseau (colonne *Length*), soit 4 (boucle locale) + 40 (IPv6) + 20 (TCP) + 658 (HTTP). La réponse fait 704 octets, soit 64 + 640. Chaque couche ajoute bien son en-tête autour de la précédente.

`localhost` est résolu en `::1` et non en `127.0.0.1` : Apache écoute donc aussi en IPv6 (le navigateur affiche d'ailleurs `Remote Address : [::1]:80`).

### Étape 5 : La connexion TCP

| N° | Temps (s) | Sens | Information |
| --- | --- | --- | --- |
| 8996 | 58,7537 | client → serveur | `[SYN]` Seq=0 Win=65535 MSS=65475 WS=256 SACK_PERM |
| 8997 | 58,7539 | serveur → client | `[SYN, ACK]` Seq=0 Ack=1 |
| 8998 | 58,7539 | client → serveur | `[ACK]` Seq=1 Ack=1 |
| 8999 | 58,7545 | client → serveur | `GET /test_http/formulaire.php HTTP/1.1` (658 octets) |
| 9000 | 58,7546 | serveur → client | `[ACK]` Ack=659 |
| 9014 | 58,8365 | serveur → client | `HTTP/1.1 200 OK (text/html)` (640 octets) |
| 9015 | 58,8366 | client → serveur | `[ACK]` Ack=641 |
| 9792 | 63,8521 | serveur → client | `[FIN, ACK]` Seq=641 Ack=659 |
| 9793 | 63,8522 | client → serveur | `[ACK]` Seq=659 Ack=642 |

**Schéma du dialogue client-serveur :**

```
 Client (navigateur)                             Serveur (Apache)
   ::1, port 6716                                  ::1, port 80
          |                                              |
          |-- 1. [SYN]  Seq=0 -------------------------->|
          |<-- 2. [SYN, ACK]  Seq=0  Ack=1 --------------|
          |-- 3. [ACK]  Seq=1  Ack=1 ------------------->|
          |                                              |
          |            Connexion TCP établie             |
          |                                              |
          |-- GET /test_http/formulaire.php  (658 o) --->|
          |<-- [ACK]  Ack=659 ---------------------------|
          |<-- HTTP/1.1 200 OK  (640 o) -----------------|
          |-- [ACK]  Ack=641 --------------------------->|
          |                                              |
          |   5 s sans activité (Keep-Alive timeout=5)   |
          |                                              |
          |<-- [FIN, ACK]  Seq=641  Ack=659 -------------|
          |-- [ACK]  Seq=659  Ack=642 ------------------>|
          |                                              |
```

**Analyse :**

- **Poignée de main :** les trois paquets `[SYN]`, `[SYN, ACK]`, `[ACK]` précèdent la requête HTTP. Entre le SYN et le SYN-ACK s'écoulent environ 0,16 ms, et les trois paquets prennent 0,22 ms au total : la latence d'une boucle locale est négligeable.
- **Temps de traitement :** il s'écoule environ **82 ms** entre la requête (n° 8999) et la réponse (n° 9014). Ce délai est celui d'Apache et de PHP, pas du réseau.
- **Fermeture :** le `[FIN, ACK]` du serveur arrive environ **5 secondes** après la réponse (63,85 s − 58,84 s), ce qui correspond exactement à `Keep-Alive: timeout=5` vu dans les en-têtes. C'est le serveur qui ferme la connexion après 5 s d'inactivité.
- **Fin de vie de la connexion :** le navigateur n'envoie pas de `FIN` de son côté. On voit ensuite des paquets *TCP Keep-Alive* toutes les 45 s environ (108,87 s et 153,88 s), puis un `[RST, ACK]` du serveur à 183,85 s, soit 2 minutes après le `FIN`.
- **Cohérence avec le navigateur :** la réponse fait 640 octets de données TCP, soit 255 octets d'en-têtes et 385 octets de corps, ce qui correspond à `Content-Length: 385` et aux « 640 B transferred » de l'onglet Réseau.

### Étape 6 : Suivre le flux TCP

Clic droit sur la requête HTTP, *Suivre*, *Flux TCP* (filtre `tcp.stream eq 653`).

**Comparaison avec l'outil du navigateur (Partie 1) :**

- **Contenu identique :** même ligne de requête `GET /test_http/formulaire.php HTTP/1.1`, mêmes en-têtes de requête (Host, User-Agent, Accept...), même réponse `HTTP/1.1 200 OK` avec `Content-Length: 385`, `Content-Type: text/html; charset=UTF-8`, `Keep-Alive: timeout=5, max=100`.
- **Ordre des en-têtes :** Wireshark montre l'ordre réel d'émission (`Host`, `Connection`, `sec-ch-ua`..., `Accept-Encoding`, `Accept-Language` en dernier), alors que l'outil du navigateur affiche les en-têtes de requête triés par ordre alphabétique (`Accept`, `Accept-Encoding`, `Accept-Language`, `Connection`, `Cookie`, `Host`...).
- **Ce que Wireshark montre en plus :** la poignée de main TCP, les acquittements et la fermeture, invisibles dans le navigateur.

### Étapes 7 à 10 : POST en clair, DNS, HTTPS, enregistrement (voir capture)

> 
> 
> - **Étape 7** : filtre `http.request.method == "POST"` et rubrique *HTML Form URL Encoded* montrant le nom saisi en clair.
> - **Étape 8** : filtre `dns` (requête *Standard query A* et réponse pour `www.wikipedia.org`), comparée à `nslookup` .
> 
> ```
> DNS request timed out.
> timeout was 2 seconds.
> Serveur :   UnKnown
> Address:  fe80::1
> 
> Réponse ne faisant pas autorité :
> Nom :    dyna.wikimedia.org
> Addresses:  2a02:ec80:300:ed1a::1
>             185.15.59.224
> Aliases:  www.wikipedia.org
> ```
> 
> - **Étape 9** : filtre `tls` (Client Hello avec le *Server Name Indication*, paquets *Application Data* illisibles).
> - **Étape 10** : enregistrement de la capture dans `captures/tp2-http.pcapng`.

---

## Réponses aux questions

### 1. Pourquoi dit-on que HTTP est sans état ? Qu'est-ce qui permet malgré tout au serveur de reconnaître un visiteur ?

HTTP est dit **sans état** parce que chaque requête est traitée isolément : le serveur ne garde aucune mémoire des requêtes précédentes. Chaque message doit donc contenir tout ce dont le serveur a besoin. On le voit dans nos captures : la requête vers `formulaire.php` ne contient rien qui indique que l'on vient d'envoyer un nom quelques secondes plus tôt.

La connexion TCP *keep-alive* n'y change rien : c'est une optimisation de transport (réutiliser la même connexion), pas une mémoire applicative. On a d'ailleurs vu le serveur la fermer après 5 secondes.

Ce qui permet de reconnaître un visiteur, ce sont les **cookies** : le serveur envoie un en-tête `Set-Cookie`, et le navigateur renvoie ensuite `Cookie` à chaque requête. On l'a observé avec `Cookie: PHPSESSID=...` (figure 2) : cet identifiant de session permet à PHP de retrouver, côté serveur, les données de la session du visiteur. D'autres mécanismes existent aussi (jeton dans l'en-tête `Authorization`, identifiant dans l'URL).

### 2. Où se trouvent les données d'un formulaire envoyé en GET ? en POST ? L'une des deux est-elle sûre sans HTTPS ?

- **GET :** les données sont dans l'URL, dans la *query string* (`?nom=emmanuel`, figures 3 et 4). Elles sont visibles dans la barre d'adresse, enregistrées dans l'historique, les favoris et les journaux du serveur. PHP les lit dans `$_GET`.
- **POST :** les données sont dans le **corps** de la requête, encodées selon le `Content-Type` (`application/x-www-form-urlencoded`). L'URL ne change pas. PHP les lit dans `$_POST`.

**Aucune des deux n'est sûre sans HTTPS.** Le POST évite d'afficher les données dans l'URL et l'historique, mais le corps de la requête circule en clair : quiconque capture le trafic peut le lire avec Wireshark. C'est précisément ce que montre l'étape 7 de la Partie 3. Les données sensibles (mot de passe, données personnelles) doivent donc passer en **POST et en HTTPS**. Le GET reste réservé aux recherches, aux filtres et aux pages que l'on veut pouvoir mettre en favori.

### 3. Que contient la réponse à une requête `HEAD` ?

La réponse à une requête `HEAD` contient la **ligne de statut et les en-têtes seulement, sans corps** : le serveur répond comme pour un `GET`, mais n'envoie pas le contenu de la page. Pour `formulaire.php`, on obtient typiquement `HTTP/1.1 200 OK` et les en-têtes `Date`, `Server`, `X-Powered-By`, `Content-Type: text/html; charset=UTF-8`, etc., mais pas le HTML. Cela sert à vérifier qu'une ressource existe, à connaître sa taille, son type ou sa date de modification sans la télécharger.

Commande de test : `curl.exe -I http://localhost/test_http/formulaire.php`

```jsx
HTTP/1.1 200 OK
Date: Mon, 23 Nov 2026 23:00:16 GMT
Server: Apache/2.4.65 (Win64) OpenSSL/3.5.1 PHP/8.3.36
X-Powered-By: PHP/8.3.36
Content-Type: text/html; charset=UTF-8
```

### 4. Combien de paquets faut-il pour ouvrir une connexion TCP ? À quoi sert chacun ?

Il faut **trois paquets** (la poignée de main en trois temps), observés dans Wireshark aux n° 8996 à 8998 :

1. **`[SYN]`** (client → serveur) : le client demande l'ouverture de la connexion et propose ses paramètres (numéro de séquence initial, `MSS=65475`, facteur de fenêtre `WS=256`, `SACK_PERM`).
2. **`[SYN, ACK]`** (serveur → client) : le serveur accepte, acquitte le SYN du client (`Ack=1`) et envoie ses propres paramètres.
3. **`[ACK]`** (client → serveur) : le client acquitte à son tour (`Ack=1`). La connexion est établie et la requête HTTP peut partir (n° 8999).

Cette poignée de main sert aussi de diagnostic : un `[SYN]` sans réponse signale un serveur arrêté ou un pare-feu, un `[RST]` en réponse signale que rien n'écoute sur ce port.

### 5. Quelles informations Wireshark montre-t-il pour un site en HTTPS, et lesquelles cache-t-il ?

**Visible :**

- les adresses IP source et destination, et le port (443) ;
- la poignée de main TCP ;
- la poignée de main TLS, dont le **Client Hello** qui contient le nom du site visité (*Server Name Indication*) ;
- la taille et le rythme des échanges (volume, horaires des paquets).

**Caché (chiffré) :**

- l'URL complète (chemin et paramètres) ;
- les en-têtes HTTP (dont les cookies) ;
- le contenu des pages ;
- les données des formulaires (mots de passe compris).

Ces éléments apparaissent dans des paquets *Application Data* illisibles. C'est l'inverse de notre capture en HTTP, où tout était lisible : méthode, chemin, en-têtes, corps. Pour déboguer une application en HTTPS, Wireshark ne suffit plus : on utilise les outils du navigateur, qui voient les données avant le chiffrement.

---

## Synthèse

- **Le navigateur** montre l'échange décodé et confortable à lire (en-têtes, paramètres, statut, durée) : c'est le premier outil de débogage.
- **Wireshark** montre ce qui circule réellement : couches empilées (boucle locale, IPv6, TCP, HTTP), poignée de main, acquittements, fermeture de la connexion.
- Les deux vues concordent : 385 octets de corps, 255 octets d'en-têtes, 640 octets au total, `Keep-Alive: timeout=5` retrouvé dans le `FIN` envoyé 5 secondes après la réponse.
- En HTTP clair, tout est lisible par quiconque capture le trafic : d'où la règle **HTTPS partout**.
- Savoir à quel niveau regarder (navigateur, journaux serveur, Wireshark) permet de situer une panne au lieu de modifier le code au hasard.

---