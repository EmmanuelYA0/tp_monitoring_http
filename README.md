# TP : Observer HTTP, du navigateur à Wireshark

Ce dépôt contient le travail du TP « Observer HTTP » du cours Architectures et applications web. Il s'agit d'observer un même échange HTTP à deux niveaux, sur un serveur Apache/PHP local : d'abord dans l'onglet Réseau du navigateur (GET, POST, erreur 404, redirection 302, cache), puis dans Wireshark sur l'interface de boucle locale (couches réseau, poignée de main TCP, flux TCP).

Le dépôt contient `formulaire.php` (page de test qui reçoit un nom en GET ou en POST), `redirige.php` (redirection 302 vers `formulaire.php`), le dossier `captures/` (captures d'écran et `tp2-http.pcapng`) et `compte-rendu-monitoring-httpd.md` (tableaux remplis, schéma de la poignée de main TCP, réponses aux questions). Pour tester, placer les deux fichiers PHP dans un dossier du serveur local (par exemple `C:\xampp\htdocs\test_http`), puis ouvrir `http://localhost/test_http/formulaire.php`.
