# Plateforme robotique 3D — architecture et guide (type The Construct)

Oct 13, 2026 · @adam

# Partie 0 — Vue d'ensemble en schémas

Cette partie résume toute l'architecture en images : qui parle à qui, par quel canal, avec quel langage, et où on tape chaque commande. À lire en premier, avant les chapitres détaillés.

## 0.0 Cadre imposé par le cours : dates, grille, pile technique

Le cours impose quatre dates de rendu et une grille de 100 points qui note l'UML et une application Java Spring Boot ; tout le plan de ce document est calé dessus. Trois décisions en découlent : l'application est codée en **Java (Spring Boot)**, Python ne sert que dans l'environnement d'exercice (nœuds ROS 2, code des étudiants), et le périmètre est réduit à un MVP.

**Dates de rendu**

| Date | Semaine ISO | Rendu | Où |
| --- | --- | --- | --- |
| 01/11/2026 | S44 | CdC initial, à valider avec le client, pas à rendre | avec le client |
| 30/11/2026 | fin S48 | CdC final | Arche |
| 20/01/2027 | S3 | documentation technique, application finale (sources et tests), guide d'utilisation | Arche |
| à confirmer | S3 ou S4 | démo-concours | sur place |

**Du plan en 16 semaines au calendrier réel.** Il reste 13 semaines de travail, car S52 et S53 sont des vacances. Les chapitres 7, 20 et 22 à 26 comptent en semaines de projet : la semaine 1 est S42, la semaine 10 est S51, la semaine 11 est S1 de 2027, la semaine 13 est S3. Les semaines 14 à 16 du plan initial n'existent plus.

| Phase technique (chapitre 7) | Semaines projet | Semaines ISO | Phase du cours |
| --- | --- | --- | --- |
| 1. Contrats + tranche verticale | 1 à 2 | S42 à S43 | analyse |
| 2. LiDAR, odométrie, TF | 3 à 4 | S44 à S45 | analyse puis conception |
| 3. Un conteneur par session | 5 à 6 | S46 à S47 | conception |
| 4. Éditeur, terminal, vue 3D | 7 à 8 | S48 à S49 | conception puis production |
| 5. Score et premières leçons (le SLAM devient optionnel) | 9 à 10 | S50 à S51 | production |
| 6. 2 à 3 leçons complètes | 11 à 12 | S1 à S2 | déploiement |
| 7-8. Sécurité, tests, documents de rendu, démo | 13 | S3 | déploiement |

Les phases 1 à 4 se font pendant l'analyse et la conception : ce sont des preuves de faisabilité (un robot qui bouge, affiché en 3D) qui servent aussi à écrire un CdC réaliste. Le gel des fonctionnalités tombe à la fin de la semaine 12 (16/01) ; il reste trois jours ouvrés avant le rendu, d'où l'importance d'écrire la documentation au fil de l'eau.

**Périmètre du MVP**

| Dans le MVP | Reporté ou optionnel |
| --- | --- |
| 1 robot, LiDAR 2D, vue 3D Three.js | SLAM (`slam_toolbox`) et carte |
| Éditeur Monaco + terminal, code Python des exercices | C++ et paquet `robot_edu_cpp` |
| Sessions isolées, un conteneur par étudiant | Rôle administrateur, tableaux de bord Grafana |
| 1 cours, 2 à 3 exercices notés, score | Plusieurs langues, comptes avancés |

**Grille d'évaluation et où chaque critère est traité** (la partie 4 de ce document couvre les critères 1 à 13)

| N° | Critère | Points | Traité en | Responsable |
| --- | --- | --- | --- | --- |
| 1 | Analyse fonctionnelle | 5 | 4.1 | D |
| 2 | Diagramme de cas d'utilisation | 5 | 4.2 | D |
| 3 | Diagramme de classes (10 entités JPA) | 10 | 4.3 | C |
| 4 | Diagrammes de séquence | 5 | 4.4 et 0.2 | C et A |
| 5 | Diagrammes d'activité | 5 | 4.5 | B |
| 6 | Structure Spring Boot (model, repository, service, controller) | 5 | 4.6 | C |
| 7 | Entités JPA | 10 | 4.3 et 4.7 | C et D |
| 8 | Repositories | 5 | 4.7 | C et D |
| 9 | Services métier | 10 | 4.7 | C et D |
| 10 | Contrôleurs REST | 10 | 4.8 | C et D |
| 11 | Tests fonctionnels (Postman, Swagger) | 5 | 4.9 | D |
| 12 | Documentation technique | 10 | 4.10 | tous, D coordonne |
| 13 | Qualité du code | 5 | 4.11 | tous |
| 14 | Autonomie et initiative (interface web, validation, pagination) | 5 | l'interface React et la 3D | B |
| 15 | Présentation orale | 5 | 4.12 | tous |

**Points à faire confirmer par l'enseignant**

- Le simulateur `sim_node` est en Python (PyBullet) ; on le considère comme l'environnement d'exercice, pas comme l'application.
- La consigne cite H2 ou MySQL ; nous prévoyons PostgreSQL (JPA reste identique). H2 sert aux tests automatiques.
- La date et le format de la démo-concours.
- Le plan UML : un outil imposé (EA est cité pour le modèle de base de données) ou libre.

**Nouvelle répartition du Java.** Comme 40 points portent sur le code Spring Boot, il est partagé : C porte `auth`, `users`, `sessions` et la sécurité ; D porte `catalog`, `content`, `progress` et `grading` ; B porte `workspace` ; A porte le `realtime-proxy` (il connaît le protocole rosbridge). Les dossiers de la partie 3 décrivent les tâches ; en cas de conflit, cette répartition fait foi.

## 0.1 Carte globale : qui parle à qui

Le navigateur ne parle qu'à la gateway ; tout le reste est derrière elle. Chaque étudiant obtient son propre conteneur, créé par un seul composant, le gestionnaire de sessions.

&#91;embedded content: architecture globale · blocs, connexions et propriétaires A à D\]

**Comment lire le schéma.** Une flèche va de celui qui appelle vers celui qui répond ; une flèche à deux pointes est un échange dans les deux sens. La couleur donne le propriétaire du bloc (A à D, voir le chapitre 22). Les pointillés marquent ce qui est écrit par l'étudiant de la plateforme, pas par l'équipe.

### Rôle de chaque bloc

| Bloc | Rôle | Technologie | Parle à | Propriétaire |
| --- | --- | --- | --- | --- |
| Navigateur | Affiche le site, la vue 3D, l'éditeur et le terminal. Ne parle jamais directement à ROS ni à la base de données | React, TypeScript, Three.js, Monaco, xterm.js | gateway | B |
| gateway | Point d'entrée unique : certificat HTTPS, fichiers du site, aiguillage de `/api` et `/ws` | Caddy | api, realtime-proxy | D |
| api | Comptes, catalogue, leçons, espace de travail, sessions, tentatives, progression | Spring Boot (Java) | PostgreSQL, Redis, session-manager | C |
| realtime-proxy | Vérifie chaque connexion WebSocket : jeton, liste blanche de topics, limites de débit | Spring Boot (Java), WebSocket | conteneur de session, session-manager | C |
| session-manager | Crée, surveille et supprime les conteneurs ; seul composant qui parle à Docker | Spring Boot (Java) + docker-java | api, Docker Engine, realtime-proxy | C |
| PostgreSQL | Données durables : comptes, cours, sessions, scores | PostgreSQL 16 | api, grading | C |
| Redis Streams | Bus d'événements entre services (tentative soumise, session arrêtée…) | Redis 7 | api, grading, session-manager | C |
| grading | Calcule le score à partir de l'état réel de la simulation, jamais d'une valeur envoyée par le client | Spring Boot (Java), consommateur d'événements | Redis, conteneur de session | A (avec C) |
| Docker Engine | Fait tourner les conteneurs avec leurs limites de mémoire, de CPU et de réseau | Docker | session-manager | D |
| Conteneur de session | Un environnement ROS 2 isolé par étudiant, détruit à la fin | Image `robot-edu/runtime` (ROS 2 Jazzy) | proxy, grading | A |
| `sim_node` | Simule le robot, le monde et le LiDAR ; publie les topics | Python, PyBullet, rclpy | topics ROS 2 | A |
| rosbridge | Traduit les topics ROS 2 en messages JSON par WebSocket | `rosbridge_server` | realtime-proxy | A |
| ttyd | Terminal web de la session | ttyd | realtime-proxy | A |
| Code étudiant | Le node que l'étudiant écrit pendant la leçon | Python (rclpy) ou C++ (rclcpp) | topics ROS 2 | l'étudiant |

### Les connexions : canal, contenu et protection

| Connexion | Canal | Ce qui circule | Protection |
| --- | --- | --- | --- |
| Navigateur → gateway | HTTPS | Pages du site, appels REST en JSON | TLS, cookies `HttpOnly`, limitation de débit |
| Navigateur ↔ gateway ↔ proxy | WSS (WebSocket sécurisé) | Messages rosbridge (JSON) et entrées du terminal | Jeton court lié à une session, vérifié au premier message |
| gateway → api | HTTP interne | Requêtes `/api/v1/...` | Réseau Docker interne |
| api → session-manager | HTTP interne | Créer, lire, supprimer une session | Clé interne `INTERNAL_API_KEY` |
| session-manager → Docker Engine | API Docker | Créer et supprimer des conteneurs | Seul composant autorisé ; paramètres en liste blanche |
| proxy ↔ conteneur | WebSocket interne (ports 9090 et 7681) | JSON rosbridge, terminal | Un réseau Docker par session |
| api ↔ PostgreSQL | SQL | Lignes des tables | Chaque module n'écrit que dans son schéma |
| api, grading, session-manager ↔ Redis | Redis Streams | Événements `session.*`, `attempt.*`, `grading.*` | Déduplication par `eventId` |
| grading → conteneur | Sonde de lecture | Pose réelle du robot, topics enregistrés | Mécanisme exact à décider (proposition : appel interne via rosbridge) |
| nœud ROS ↔ nœud ROS | DDS (topics ROS 2) | `/cmd_vel`, `/odom`, `/scan`, `/tf`… | Même conteneur, `ROS_DOMAIN_ID` propre à la session |

## 0.2 Parcours d'une session, du clic au score

Seize messages décrivent toute la vie d'une session : le conteneur est créé à la demande, le navigateur ne le joint que par le proxy, et le score est calculé côté serveur.

&#91;embedded content: séquence · 16 messages entre 6 acteurs\]

**À retenir**

- **Étapes 1 à 5** : l'API vérifie les droits, le quota et l'idempotence, puis demande la session au manager. La session ne passe à `running` que lorsque le conteneur répond à `/healthz`, pour que le navigateur ne se connecte jamais trop tôt.
- **Étapes 6 à 8** : le navigateur interroge la session jusqu'à `running`, reçoit les adresses WebSocket et un jeton valable pour cette session seulement, puis se connecte au proxy.
- **Étapes 9 à 11** : tout passe par le proxy, qui filtre les messages. Le navigateur n'a jamais d'accès direct au conteneur.
- **Étapes 12 à 16** : le score vient de `grading`, qui lit l'état réel de la simulation ; le client ne peut pas l'imposer.
- **Non dessiné** : à l'arrêt ou à l'expiration, le manager récupère les fichiers de l'étudiant, supprime le conteneur et publie `session.stopped` ; le proxy ferme alors toutes les connexions de cette session.

Point à décider à la semaine 5 : la façon exacte dont `grading` lit la pose réelle (message 14). Proposition : un appel interne à rosbridge dans le conteneur, depuis le réseau de la session.

## 0.3 Dans le conteneur : le graphe ROS 2

Un conteneur de session fait tourner quatre nœuds ROS 2 qui échangent des messages sur un bus de topics ; rosbridge est la seule passerelle vers le navigateur. C'est le schéma à garder en tête pour A, mais aussi pour B et D, qui consomment ces topics.

&#91;embedded content: graphe ROS 2 · nœuds, topics et passerelle web\]

| Topic | Type de message | Publié par | Lu par |
| --- | --- | --- | --- |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | code étudiant, navigateur (via rosbridge) | `sim_node` |
| `/odom` | `nav_msgs/msg/Odometry` | `sim_node` | code étudiant, `slam_toolbox`, navigateur |
| `/scan` | `sensor_msgs/msg/LaserScan` | `sim_node` | code étudiant, `slam_toolbox`, navigateur |
| `/tf` | `tf2_msgs/msg/TFMessage` | `sim_node` (odom → base\_link), publieur d'état (liens fixes, roues), `slam_toolbox` (map → odom) | tous |
| `/joint_states` | `sensor_msgs/msg/JointState` | `sim_node` | publieur d'état, navigateur |
| `/clock` | `rosgraph_msgs/msg/Clock` | `sim_node` | tous (avec `use_sim_time:=true`) |
| `/map` | `nav_msgs/msg/OccupancyGrid` | `slam_toolbox` | code étudiant, navigateur |

Trois règles à ne pas oublier : (1) tous les nœuds du conteneur utilisent `use_sim_time:=true` ; (2) le simulateur est le seul à publier l'horloge ; (3) un topic absent de la liste blanche de la leçon est refusé par le proxy, même si ROS le publie.

## 0.4 Les langages et les fichiers qu'on va manipuler

On écrit dans cinq familles de fichiers. Chacune a sa façon d'être lancée, et c'est la source d'erreur n°1 pour des débutants : oublier l'étape 2.

&#91;embedded content: langages et fichiers · de la source à l'exécution\]

| Fichier | Rôle | Après une modification |
| --- | --- | --- |
| Python (`.py`) | nœuds ROS 2, code des étudiants, `sim_node` | relancer le nœud ; avec `--symlink-install`, pas de recompilation |
| C++ (`.cpp`, `CMakeLists.txt`) | nœuds performants, leçons avancées | `colcon build` puis relancer |
| Xacro (`.xacro`) | description du robot avec macros et variables | `make urdf` (génère l'URDF), puis relancer |
| URDF (`.urdf`) | robot « à plat », lu par PyBullet, `robot_state_publisher` et le navigateur | fichier généré : ne jamais l'éditer à la main |
| Launch (`.launch.py`, `.launch.xml`) | démarre plusieurs nœuds avec leurs paramètres | `ros2 launch` suffit |
| YAML / JSON | paramètres, définition des leçons, scènes, scoring | relire au démarrage |

**Paquets ROS 2 par type de build**

| Paquet | Type | Contenu | Responsable |
| --- | --- | --- | --- |
| `robot_edu_sim` | `ament_python` | `sim_node`, capteurs, monde | A |
| `robot_edu_description` | `ament_python` (données) | xacro, URDF généré, meshes | A |
| `robot_edu_bringup` | `ament_python` (données) | launch files, YAML | A |
| `robot_edu_cpp` (nouveau, optionnel) | `ament_cmake` | exemples C++ pour les leçons avancées | A, relu par D |

Décisions à retenir :

- `sim_node` reste en Python, car PyBullet est utilisé depuis Python.
- Le xacro est développé **une seule fois** en URDF (`make urdf` en dev, étape du Dockerfile du runtime), et cet unique URDF sert à PyBullet, à `robot_state_publisher` et à `urdf-loader` côté navigateur : une seule source de vérité.
- Le C++ demande une compilation, plus lente et gourmande en mémoire dans un conteneur limité à 768 Mo : on commence par Python et on introduit le C++ dans une leçon ultérieure.
- À reporter ensuite dans le tableau du chapitre 13.1 et dans l'arborescence du chapitre 15 : le paquet `robot_edu_cpp`.

## 0.5 Où on travaille : IDE, terminal de lancement, navigateur

Il y a deux contextes. En **développement**, l'équipe édite avec son IDE sur WSL et lance tout dans un conteneur Docker Compose qui voit le dossier par un volume. En **session**, l'étudiant édite dans Monaco et lance dans son conteneur isolé.

&#91;embedded content: boucle de travail · mode développement et mode session\]

**Commandes à connaître en développement** (dans le terminal du conteneur, sauf mention) :

| Commande | À quoi ça sert |
| --- | --- |
| `docker compose up -d` (sur WSL) | démarre la pile de dev |
| `docker compose exec ros bash` (sur WSL) | ouvre un terminal dans le conteneur ROS |
| `make urdf` | transforme le xacro en URDF |
| `colcon build --symlink-install` | compile les paquets ; après le premier build, les `.py` modifiés sont pris sans recompiler |
| `source install/setup.bash` | rend les paquets visibles dans ce terminal |
| `ros2 launch robot_edu_bringup sim.launch.py` | lance simulateur, publieur d'état et rosbridge |
| `ros2 topic list` / `ros2 topic echo /odom` | vérifier ce qui circule |

**Choix d'IDE (à vérifier par l'équipe)** : IntelliJ IDEA est l'outil naturel pour le Java et Spring Boot (l'édition Ultimate est gratuite avec la licence étudiante JetBrains) ; VS Code, avec ses extensions et Dev Containers, couvre bien le web, le Python, le C++ et le XML du runtime. Le choix peut être mixte, à condition que tout le monde utilise les mêmes commandes dans le conteneur.

## 0.6 Récapitulatif : dossier du dépôt → image → processus → responsable

Ce tableau relie chaque dossier du monorepo à ce qu'il devient en production. Quand on ne sait plus où ranger un fichier, c'est ici qu'on regarde.

| Dossier du dépôt | Image Docker | Processus qui tourne | Responsable |
| --- | --- | --- | --- |
| `frontend/` | `robot-edu/web` (statique, servi par Caddy) | React + Vite compilé | B |
| `backend/api/` | `robot-edu/api` | Spring Boot (couches model, repository, service, controller) | C |
| `services/session-manager/` | `robot-edu/session-manager` | seul client de Docker | C |
| `backend/realtime-proxy/` | `robot-edu/realtime-proxy` | proxy WebSocket, liste blanche, limites | C |
| `backend/grading/` | `robot-edu/grading` | worker qui note les tentatives | D (avec A pour la lecture de pose) |
| `ros/ros2_ws/src/robot_edu_sim` | `robot-edu/runtime` | `sim_node` (PyBullet) | A |
| `ros/ros2_ws/src/robot_edu_description` | `robot-edu/runtime` | URDF généré, meshes | A |
| `ros/ros2_ws/src/robot_edu_bringup` | `robot-edu/runtime` | `ros2 launch` : nœuds + rosbridge + ttyd | A |
| `ros/ros2_ws/src/robot_edu_cpp` (optionnel) | `robot-edu/runtime` | exemples C++ compilés | A |
| `content/` | volume lu par l'API | leçons, exercices, scènes en YAML/JSON | D |
| `infra/` | n/a | Compose, Caddy, CI GitLab | D |

Les lettres A, B, C, D sont celles de la répartition de la partie 3 ; chacun peut les remplacer par son prénom.

## 1. Vision et périmètre

Le projet est une plateforme en ligne où l'étudiant apprend ROS 2 en écrivant du code, en le lançant sur un robot simulé en 3D et en voyant le résultat dans son navigateur, comme sur The Construct. La simulation tourne sans Gazebo : un moteur physique léger (PyBullet) fait tourner le robot côté serveur, et Three.js l'affiche.

**Ce que fait l'étudiant**

1. Choisit un cours et une leçon (niveau débutant à intermédiaire).
2. Lit la leçon, avec le code d'exemple à côté.
3. Écrit son node ROS 2 dans l'éditeur du navigateur.
4. Lance sa session : un environnement ROS isolé démarre, avec le robot dans un monde 3D.
5. Exécute son code dans un terminal web et voit le robot bouger en 3D, avec le scan LiDAR, la trajectoire et la carte.
6. Soumet son résultat, reçoit un score et retrouve sa progression.

**Livré en 4 mois (MVP)**

- 1 robot différentiel type TurtleBot, décrit en URDF.
- 2 à 3 mondes 3D : pièce, couloir, labyrinthe.
- LiDAR 3D par lancer de rayons, odométrie, TF ; caméra simple si le temps le permet.
- Vue 3D Three.js + éditeur de code + terminal web dans une même page.
- 3 à 5 leçons complètes, de `/cmd_vel` jusqu'à la carte et à un SLAM sur la simulation.
- Sessions isolées, avec progression et score enregistrés.

**Hors périmètre**

- Physique de contact avancée, matériaux, capteurs optiques réalistes, bras articulés, multi-robot.
- Un SLAM écrit de zéro par l'équipe : on configure et on rejoue un SLAM existant.
- Kubernetes, comptes enseignants, paiement, certificats.

Le simulateur est honnête sur ses limites : c'est une simulation pédagogique, pas un banc de validation de robot réel.

## 2. Architecture

La plateforme se compose de sept services permanents (web, api, base de données, Redis, gestionnaire de sessions, proxy et grading) et d'un conteneur créé à la demande pour chaque étudiant. Le simulateur, rosbridge et le terminal tournent à l'intérieur de ce conteneur, ce qui évite les problèmes de découverte DDS entre conteneurs.

&#91;embedded content: architecture · navigateur, services, conteneur de session\]

Le navigateur appelle l'API en HTTPS pour créer une session. Le gestionnaire de sessions crée alors le conteneur ; ensuite, le navigateur ne le joint que par le proxy WebSocket, qui vérifie le jeton de session.

**Services de Docker Compose en développement** : `web`, `api`, `db`, `redis`, `session-manager`, `proxy` et `grading`. L'image `robot-sim` (ROS 2 Jazzy + PyBullet + rosbridge) est construite par Compose mais lancée uniquement par le gestionnaire de sessions, une fois par étudiant. Le détail de chaque service est au chapitre 10.

## 3. Simulation 3D sans Gazebo

La simulation est un node ROS 2 Python qui embarque PyBullet en mode sans interface graphique. Il remplace Gazebo : même contrat côté ROS (topics, TF, temps simulé), mais sans rendu côté serveur, donc sans GPU.

**Ce que fait le node `sim_node`**

- Charge l'URDF du robot et la description du monde (fichier JSON de murs et d'obstacles).
- Avance la physique à pas fixe (240 Hz par défaut dans PyBullet) pour que le résultat soit reproductible.
- Lit `/cmd_vel`, le convertit en vitesses de roues et les applique aux articulations.
- Lance des rayons pour le LiDAR avec `rayTestBatch` : un scan 2D pour `/scan`, ou plusieurs couches pour un nuage de points 3D.
- Publie `/clock` : tous les nodes de l'étudiant tournent avec `use_sim_time:=true`, ce qui est indispensable pour le SLAM.

**Topics du simulateur**

| Topic | Type | Cadence indicative | Sens |
| --- | --- | --- | --- |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | selon le code étudiant | étudiant → simulateur |
| `/odom` | `nav_msgs/msg/Odometry` | 30 Hz | simulateur → étudiant et vue 3D |
| `/scan` | `sensor_msgs/msg/LaserScan` | 10 Hz | simulateur → étudiant et vue 3D |
| `/points` | `sensor_msgs/msg/PointCloud2` | 5 Hz, si LiDAR 3D | simulateur → vue 3D |
| `/joint_states` | `sensor_msgs/msg/JointState` | 30 Hz | simulateur → vue 3D (roues) |
| `/tf` | `tf2_msgs/msg/TFMessage` | 30 Hz | simulateur → tous |
| `/clock` | `rosgraph_msgs/msg/Clock` | 50 Hz | simulateur → tous |

**Une seule source de vérité pour le monde.** Le même fichier JSON décrit les murs pour PyBullet (collisions et rayons) et pour Three.js (affichage). Ainsi, ce que l'étudiant voit est exactement ce que le LiDAR mesure.

**Le robot.** Un URDF de robot différentiel (2 roues motrices, 1 roulette, 1 capteur LiDAR) avec des formes simples au début. Le même URDF est servi au navigateur, où `urdf-loader` le charge dans Three.js.

**Limites à afficher dans les leçons.** Le LiDAR simulé est un lancer de rayons parfait, sans bruit ni réflexion réaliste, et le contact des roues est simplifié. On peut ajouter du bruit gaussien sur le scan et l'odométrie comme exercice, ce qui rend le SLAM plus instructif.

## 4. Interface web : vue 3D, éditeur et terminal

La page de leçon affiche côte à côte le texte de la leçon, un éditeur de code, un terminal et la vue 3D. C'est cette disposition qui donne l'effet The Construct.

| Brique | Bibliothèque | Rôle |
| --- | --- | --- |
| Vue 3D | Three.js + `urdf-loader` | Affiche le monde, le robot URDF animé, le scan, la trajectoire et la carte |
| Connexion ROS | `roslibjs` (rosbridge) | Abonnement aux topics autorisés depuis le navigateur |
| Éditeur | Monaco (celui de VS Code) | Édition du code Python de l'étudiant, sauvegardé dans sa session |
| Terminal | xterm.js + WebSocket | Shell dans le conteneur de l'étudiant : `ros2 run`, `ros2 topic echo` |
| Application | React + TypeScript (Vite) | Cours, leçons, session, progression |

**Organisation de la vue 3D**

1. `RobotScene` : renderer, scène, lumières, caméra libre et vue de dessus, grille et axes.
2. `WorldLayer` : construit les murs et obstacles à partir du même JSON que le simulateur.
3. `RobotModel` : charge l'URDF, applique `/tf` pour la pose et `/joint_states` pour faire tourner les roues.
4. `ScanLayer` : transforme chaque `LaserScan` en points ; un nuage `PointCloud2` utilise la même couche avec des points XYZ.
5. `TrailLayer` : trajectoire issue de `/odom`.
6. `MapLayer` : `OccupancyGrid` affichée comme un plan texture.

**Règles de rendu**

- Un seul module convertit les repères ROS vers Three.js (x = x, y = z, z = −y) ; aucun autre composant ne le refait.
- Le rendu tourne à son propre rythme (60 images par seconde au plus) ; les messages ROS mettent à jour un état, ils ne déclenchent pas chacun un dessin.
- Un point de scan se calcule ainsi : angle = angle\_min + i × angle\_increment, puis x = r cos(angle), y = r sin(angle) dans le repère du capteur, et on ignore NaN, infini et les valeurs hors de range\_min et range\_max.
- Un badge indique la source des données : simulation, dataset rejoué ou robot réel.

**Leçons.** Chaque leçon est un fichier Markdown versionné (objectif, prérequis, étapes, code de départ, critères de réussite), affiché dans la page avec un bouton « Lancer la simulation ».

## 5. Backend, sessions et sécurité

Le point le plus sensible du projet est le terminal web : il donne un shell à l'étudiant. Il doit donc tourner dans un conteneur isolé, jamais sur l'hôte.

**API (Spring Boot + PostgreSQL)**

```text
GET    /health
GET    /api/v1/courses
GET    /api/v1/lessons/{lessonId}
POST   /api/v1/sessions          crée une session (leçon, monde, robot)
GET    /api/v1/sessions/{id}     état, expiration
DELETE /api/v1/sessions/{id}     arrêt et nettoyage (idempotent)
POST   /api/v1/attempts          soumission et score
GET    /api/v1/progress/me
```

**Une session = un conteneur.** Pour ressembler à The Construct, chaque étudiant obtient son propre conteneur ROS 2 avec le simulateur, rosbridge et un terminal. Le conteneur est créé par un petit service « gestionnaire de sessions ». Lui seul parle à Docker ; ni l'API ni le navigateur n'y ont accès.

**Règles de sécurité à tenir dès le MVP**

- Conteneur non privilégié, utilisateur non-root, sans accès au socket Docker, avec limites de mémoire, de CPU, de processus et de durée.
- Un réseau Docker par session ; le seul port exposé est un proxy WebSocket authentifié par un jeton court, valable pour cette session uniquement.
- Rosbridge et le terminal ne sont jamais publiés directement sur Internet.
- Liste blanche des topics que le navigateur peut lire ou publier.
- Expiration automatique de la session, puis suppression du conteneur et de ses fichiers.
- Aucun secret dans GitLab : fichiers `.env` ignorés, `.env.example` versionné.

**Comptes.** Pour la démo, des comptes locaux simples (e-mail + mot de passe haché, ou simple pseudo) suffisent. OIDC, rôles enseignant et administrateur viennent après le MVP.

## 6. Contrats et repères à figer en semaine 1

Deux fichiers versionnés dans le dépôt, relus par les quatre personnes avant d'écrire du code d'intégration : `docs/contracts.md` et `docs/frames.md`.

**`contracts.md`** liste, pour chaque interface : nom, type, cadence, responsable, paramètres, erreurs possibles et un exemple de message. Le tableau de la section 3 en est la base ; on y ajoute :

| Interface | Contenu à figer | Responsable |
| --- | --- | --- |
| Fichier monde (JSON) | murs, obstacles, point de départ, taille | A écrit, B lit |
| URDF du robot | liens, articulations, position du LiDAR | A |
| API de sessions | routes ci-dessus + codes d'erreur | C |
| Protocole du terminal web | format des messages WebSocket | C et A |
| Fichier de leçon (Markdown) | métadonnées, étapes, critères de réussite | D |

**`frames.md`** fixe la chaîne `map` → `odom` → `base_link` → `laser`, l'unité (mètre, radian), le sens positif des angles et la conversion vers Three.js. Le simulateur publie `odom` → `base_link` ; le SLAM publie `map` → `odom`.

Un changement de contrat passe par une Merge Request relue par la personne qui en dépend.

## 7. Répartition et feuille de route

| Personne | Responsable de | Livrable clé |
| --- | --- | --- |
| A | Simulateur PyBullet, URDF, mondes, nodes ROS 2, SLAM sur la simulation | `sim_node` qui répond à `/cmd_vel` et publie `/odom`, `/scan`, `/tf` |
| B | Interface : vue Three.js, éditeur Monaco, terminal xterm.js | Page de leçon complète : texte, code, terminal, vue 3D |
| C | API, base de données, gestionnaire de sessions, proxy WebSocket, sécurité | Session créée, isolée puis nettoyée à l'expiration |
| D | Leçons, images Docker et Compose, CI GitLab, tests d'acceptation | 3 à 5 leçons vérifiables et une installation depuis zéro |

Cette répartition équilibre la charge : A porte la simulation seule, et D prend le Docker et les tests au lieu de s'occuper uniquement du contenu.

&#91;embedded content: planning S42 à S4 · phases du cours, plan technique, rendus\]

Chaque phase finit par une démonstration commune, pas par des composants séparés. La première, avant la fin de la semaine 2 (S43), est une tranche verticale : A déplace un robot dans le simulateur, B l'affiche dans Three.js, C fournit une session simulée et D écrit la procédure de vérification. Après la semaine 12 (S2, fin le 16/01), plus de nouvelle fonction : la semaine 13 (S3, rendu le 20/01) va aux défauts, à la sécurité, aux documents de rendu et à la démo. Intégration sur `main` chaque semaine, relue par un binôme.

## 8. Risques, décisions ouvertes et premiers pas

**Risques principaux**

| Risque | Parade |
| --- | --- |
| PyBullet s'installe mal sur Python 3.12 (ROS 2 Jazzy, Ubuntu 24.04) | Tester `pip install pybullet` le jour 1 ; sans wheel, compiler avec `build-essential` |
| Terminal web = shell ouvert à des étudiants | Conteneur non privilégié, sans socket Docker, avec quotas et expiration (section 5) |
| Latence ou débit de rosbridge sur les scans et nuages de points | Limiter l'affichage à 10 Hz pour le scan et à 5 Hz pour le nuage ; réduire le nombre de rayons |
| Le SLAM ne converge pas sur le monde simulé | Utiliser `use_sim_time`, ajouter un peu de bruit, tester tôt avec `slam_toolbox` |
| Découverte DDS entre conteneurs sous WSL | Garder simulateur, rosbridge et CLI dans le même conteneur par session |
| A devient le goulot | Revue croisée chaque semaine, Docker et tests portés par D |

**Décisions à prendre par l'équipe**

- [ ] Moteur physique : PyBullet (recommandé) ou MuJoCo.
- [ ] Algorithme SLAM de la leçon : `slam_toolbox` ou une grille d'occupation écrite par l'équipe.
- [ ] LiDAR du MVP : 2D (`LaserScan`) seulement, ou aussi 3D (`PointCloud2`).
- [ ] Gestionnaire de sessions : service Java (Spring Boot) avec docker-java, ou scripts simples au début.
- [ ] Comptes : pseudo simple ou e-mail + mot de passe.
- [ ] Capacité du serveur : nombre de sessions simultanées à montrer en démo.

**Premiers pas de la semaine 1**

1. Créer le projet GitLab, protéger `main`, ouvrir un ticket par tâche.
2. Écrire `docs/contracts.md` et `docs/frames.md` à quatre.
3. Faire tourner ROS 2 Jazzy dans Docker (`talker` et `listener`), puis installer PyBullet dans l'image.
4. Vérifier que PyBullet fonctionne sans interface avec ce test minimal :

```python
import pybullet as p, pybullet_data
p.connect(p.DIRECT)
p.setAdditionalSearchPath(pybullet_data.getDataPath())
p.setGravity(0, 0, -9.81)
p.loadURDF('plane.urdf')
robot = p.loadURDF('husky/husky.urdf', [0, 0, 0.2])
for _ in range(240):
    p.stepSimulation()
print(p.getBasePositionAndOrientation(robot))
```

5. Afficher dans une page Three.js un cube qui bouge avec des valeurs de test, puis le brancher sur rosbridge.
6. Préparer la tranche verticale de la fin de semaine 2.

# Partie 2 — Spécification détaillée

Les chapitres 9 à 21 détaillent ce qui a été posé dans les chapitres 1 à 8 : le site web, tous les microservices, les données, l'API, le simulateur, le plan du dépôt, l'infrastructure, la sécurité, les tests, le programme et le planning.

## 9. Site web : plan des pages, routes et parcours

Ce chapitre décrit le site tel que l'utilisateur le voit : chaque page, son adresse, qui peut y accéder, quelles données elle charge et de quels composants elle est faite. Il sert de référence à B (interface) et à C (API) pour s'accorder avant de coder.

### 9.1 Principes de conception

- **La leçon est l'écran principal.** Tout le reste (catalogue, tableau de bord, profil) sert à y arriver ou à en revenir. La page de leçon regroupe texte, code, terminal et vue 3D, sans changer de page.
- **Une session ne se perd jamais.** Si l'étudiant rafraîchit la page ou revient après une coupure, la page retrouve la session en cours (ou propose d'en démarrer une nouvelle) sans perdre le code écrit.
- **L'état de la simulation est toujours visible.** Un bandeau indique : en démarrage, en cours, en pause, expirée, en erreur ; avec le temps restant avant expiration.
- **Le site reste utile sans simulation.** Le catalogue, la lecture des leçons et la progression fonctionnent même quand le serveur de simulation est plein ou arrêté.
- **Source des données toujours indiquée.** Un badge « Simulation » ou « Dataset rejoué » est visible sur la vue 3D.
- **Accessibilité de base.** Navigation au clavier, contrastes suffisants, libellés sur les boutons, pas d'information portée uniquement par la couleur.

### 9.2 Carte des pages et routes

| Route | Page | Accès | Données chargées (API) | Phase |
| --- | --- | --- | --- | --- |
| `/` | Accueil : présentation, bouton « Commencer », aperçu de la simulation | Public | `GET /courses?featured=true` | MVP |
| `/courses` | Catalogue des cours, filtre par niveau et par thème | Public | `GET /courses` | MVP |
| `/courses/:courseSlug` | Fiche d'un cours : objectifs, prérequis, liste des leçons, progression | Public (progression si connecté) | `GET /courses/{slug}`, `GET /progress/me` | MVP |
| `/learn/:courseSlug/:lessonSlug` | **Page de leçon** : texte, éditeur, terminal, vue 3D | Connecté | `GET /lessons/{id}`, `POST /sessions` | MVP |
| `/playground` | Bac à sable : robot et monde au choix, sans leçon ni score | Connecté | `GET /worlds`, `GET /robots`, `POST /sessions` | MVP |
| `/login` | Connexion | Public | `POST /auth/login` | MVP |
| `/register` | Création de compte | Public | `POST /auth/register` | MVP |
| `/dashboard` | Tableau de bord : leçon à reprendre, sessions récentes, progression globale | Connecté | `GET /progress/me`, `GET /sessions?state=active` | MVP |
| `/progress` | Progression détaillée : par cours, scores, tentatives | Connecté | `GET /progress/me/details`, `GET /attempts` | MVP |
| `/profile` | Profil et préférences (langue, thème) | Connecté | `GET /users/me`, `PATCH /users/me` | MVP |
| `/sessions/:id` | Reprise d'une session existante (redirige vers la leçon) | Propriétaire | `GET /sessions/{id}` | MVP |
| `/about` | À propos, limites de la simulation, crédits et licences des datasets | Public | statique | MVP |
| `/teach` | Espace enseignant : cours, cohortes, résultats | Enseignant | `GET /teach/...` | Phase 2 |
| `/admin` | Administration : utilisateurs, quotas, sessions actives, santé des services | Admin | `GET /admin/...` | Phase 2 |
| `/404`, `/403`, `/500` | Pages d'erreur avec lien de retour | Public | aucune | MVP |

Les pages « Connecté » redirigent vers `/login?next=<adresse>` quand la session web a expiré.

### 9.3 La page de leçon en détail

C'est la page la plus complexe du site. Sur un écran large, elle se présente en quatre panneaux redimensionnables :

```text
+------------------------------------------------------------------+
| Barre : cours > leçon 3  |  ● Session en cours  12:40  | Arrêter   |
+-------------------+----------------------------------------------+
|                   |                                              |
|  TEXTE DE LA      |            VUE 3D (Three.js)                 |
|  LEÇON            |   robot, monde, scan, trajectoire, carte     |
|  (Markdown)       |   [caméra libre | dessus]  [scan][tf][map] |
|  - objectif       |                                              |
|  - étapes        +----------------------------------------------+
|  - indices        |  ÉDITEUR (Monaco)      |  TERMINAL (xterm.js) |
|  - critères       |  fichier : move.py     |  $ ros2 run ...      |
|                   |  [Enregistrer][Lancer] |                      |
+-------------------+----------------------------------------------+
| Topics : /cmd_vel ↑  /odom ↓  /scan ↓   |  Soumettre l'exercice  |
+------------------------------------------------------------------+
```

**Panneaux et règles**

| Panneau | Contenu | Comportement |
| --- | --- | --- |
| Barre haute | Fil d'Ariane, état et minuteur de session, bouton Arrêter | Le minuteur passe en orange 5 minutes avant l'expiration, avec un bouton « Prolonger » si le quota le permet |
| Texte de la leçon | Markdown rendu, étapes numérotées, blocs de code copiables, indices masqués | Les indices se dévoilent un par un ; chaque indice dévoilé est enregistré |
| Vue 3D | Scène Three.js, boutons de couches, sélecteur de caméra, badge de source | Rendu limité à 60 images par seconde ; couches activables ; clic sur un point du scan pour voir distance et angle |
| Éditeur | Fichiers de la session (Python), enregistrement automatique | Le fichier est écrit dans le conteneur ; bouton « Lancer » équivalent à `ros2 run` dans le terminal |
| Terminal | Shell du conteneur | Plusieurs onglets, coller autorisé, historique conservé pendant la session |
| Barre basse | Topics utiles et leur cadence, bouton Soumettre | Soumettre est désactivé tant que la session n'est pas en cours |

Sur petit écran, les panneaux passent en onglets (Leçon, Code, Terminal, 3D) ; la page reste utilisable pour lire, mais l'édition est recommandée sur ordinateur.

### 9.4 Parcours utilisateur principal

1. L'étudiant arrive sur `/`, clique sur « Commencer », puis choisit un cours dans `/courses`.
2. Il crée un compte ou se connecte ; il est ramené sur la fiche du cours.
3. Il ouvre la première leçon : la page charge le texte et propose « Démarrer la simulation ».
4. Au clic, le front appelle `POST /sessions`, affiche un écran de progression (« Création de l'environnement ») et interroge `GET /sessions/{id}` jusqu'à l'état `running`.
5. Le front ouvre la connexion WebSocket vers le proxy avec le jeton de session ; la vue 3D affiche le robot, le terminal affiche une invite.
6. L'étudiant écrit son code, le lance, observe le robot et les données.
7. Il clique sur « Soumettre » ; l'API déclenche la vérification, renvoie score et retours, et propose « Leçon suivante ».
8. La session est arrêtée (par lui ou par expiration) ; le conteneur est supprimé ; le code est sauvegardé pour la prochaine ouverture.

### 9.5 États à prévoir dans l'interface

| Situation | Ce que voit l'utilisateur | Action proposée |
| --- | --- | --- |
| Serveur de simulation plein | « Toutes les simulations sont occupées, position dans la file : 3 » | Attendre ou lire la leçon sans simulation |
| Session en démarrage | Écran de progression avec étapes (conteneur, simulateur, pont) | Annuler |
| Connexion WebSocket perdue | Bandeau « Connexion perdue, nouvelle tentative… », reconnexion automatique avec délai croissant | Recharger si échec après 30 s |
| Session expirée | Fenêtre « Session terminée », code sauvegardé | Relancer une session |
| Erreur de code de l'étudiant | La sortie d'erreur reste dans le terminal ; aucun message d'erreur du site | — |
| Quota de sessions atteint | « Limite de sessions simultanées atteinte » | Arrêter une session existante |
| Données invalides (NaN, frame inconnue) | Avertissement dans la vue 3D avec le topic concerné | Lien vers l'aide de la leçon |

### 9.6 Système de design et internationalisation

- **Bibliothèque de composants** : un petit système maison (boutons, champs, onglets, panneaux redimensionnables, fenêtres modales, toasts) construit sur des variables CSS ; pas de bibliothèque lourde pour garder le contrôle.
- **Jetons de design** : couleurs, espacements, tailles de police et rayons définis une seule fois (`tokens.css`), avec un thème clair et un thème sombre. Le thème sombre est important : la vue 3D et l'éditeur sont déjà sombres.
- **Langues** : français et anglais dès le MVP, avec un fichier de traductions par langue (`i18next`). Le contenu des leçons existe dans les deux langues ou retombe sur le français.
- **Raccourcis clavier dans la page de leçon** : `Ctrl+S` enregistrer, `Ctrl+Entrée` lancer le fichier, ``` Ctrl+`` basculer vers le terminal,  ```Alt+1`à`Alt+4\` aller à un panneau.
- **Performance** : chargement différé de Three.js et de Monaco (ils ne sont téléchargés que sur la page de leçon) ; le catalogue et l'accueil restent légers.

### 9.7 Règles d'authentification côté site

- Une session web par cookie `HttpOnly`, `Secure`, `SameSite=Lax`, avec un jeton d'accès court (15 minutes) et un jeton de rafraîchissement (7 jours) pour le MVP.
- Le jeton de session de simulation est un autre jeton, obtenu par `POST /sessions`, valable pour une seule session et utilisé uniquement pour les connexions WebSocket.
- La page de leçon vérifie côté serveur que la session appartient bien à l'utilisateur ; le front ne fait pas confiance à l'adresse.

## 10. Catalogue complet des microservices

La plateforme se découpe en 15 capacités, chacune avec son rôle, ses données et son interface. Ce chapitre les décrit toutes : c'est l'architecture cible. Pour que 4 personnes puissent la livrer en 4 mois, on distingue deux choses : le **découpage logique** (chaque capacité a son module, ses tables et son API interne) et le **déploiement** (plusieurs capacités tournent dans le même processus tant qu'aucune charge ne justifie de les séparer).

### 10.1 Vue d'ensemble et déploiement MVP

| # | Service | Rôle en une phrase | Déploiement MVP | Propriétaire |
| --- | --- | --- | --- | --- |
| 1 | `gateway` | Point d'entrée HTTPS, routage, limitation de débit | Processus séparé (Caddy ou Traefik) | D |
| 2 | `auth` | Comptes, mots de passe, jetons | Module de `api` | C |
| 3 | `users` | Profil, préférences, rôles | Module de `api` | C |
| 4 | `catalog` | Cours, leçons, exercices, mondes, robots | Module de `api` | C |
| 5 | `content` | Texte des leçons et fichiers associés | Module de `api` + fichiers du dépôt | D |
| 6 | `workspace` | Code de l'étudiant conservé entre les sessions | Module de `api` | C |
| 7 | `session-manager` | Crée, surveille et supprime les conteneurs de session | Processus séparé | C |
| 8 | `runtime` | Image du conteneur de session : simulateur, rosbridge, terminal | Image Docker, 1 conteneur par session | A |
| 9 | `realtime-proxy` | Proxy WebSocket authentifié vers rosbridge et terminal | Processus séparé | C |
| 10 | `progress` | Tentatives, scores, progression | Module de `api` | C |
| 11 | `grading` | Vérifie le travail soumis selon un barème | Worker séparé | A et D |
| 12 | `artifacts` | Datasets, rosbags, cartes exportées | Module de `api` + volume ou MinIO | D |
| 13 | `events` | Bus d'événements et files de tâches | Redis (Streams) | C |
| 14 | `observability` | Journaux, métriques, alertes | Compose : Prometheus + Grafana + Loki (si le temps le permet) | D |
| 15 | `admin` et `notifications` | Gestion des utilisateurs et quotas ; e-mails | Phase 2 | C |

Règle d'extraction : un module de `api` devient un service séparé seulement si (1) il a une charge ou un rythme de déploiement différent, ou (2) il porte une frontière de sécurité. Dans ce projet, seuls `session-manager`, `realtime-proxy` et `grading` répondent à ces critères dès le début.

&#91;embedded content: microservices · monolithe modulaire et 3 services séparés\]

Les sept modules de l'API partagent un seul processus et un seul PostgreSQL. Le proxy temps réel, le gestionnaire de sessions et le worker de vérification sont séparés ; ils échangent des événements par Redis Streams (non dessiné pour le proxy et le gestionnaire, qui s'y abonnent aussi).

### 10.2 Fiches de service

Chaque fiche donne : rôle, données possédées, interface, événements, dépendances, technologie et points d'attention.

#### 1. `gateway`

- **Rôle** : unique point d'entrée public. Termine le HTTPS, route `/api/*` vers l'API, `/ws/*` vers le proxy temps réel, tout le reste vers les fichiers statiques du front.
- **Données** : aucune ; certificats TLS.
- **Interface** : HTTP et WebSocket entrants ; aucune API propre.
- **Dépendances** : `api`, `realtime-proxy`, fichiers du front.
- **Technologie** : Caddy (certificats automatiques, configuration courte) ou Traefik.
- **Attention** : limitation de débit par adresse IP, taille maximale des requêtes, en-têtes de sécurité (HSTS, CSP), journal des accès sans données personnelles.

#### 2. `auth`

- **Rôle** : inscription, connexion, déconnexion, rafraîchissement de jeton, réinitialisation du mot de passe.
- **Données** : `users.credentials`, `auth.refresh_tokens`.
- **Interface** : `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`.
- **Événements émis** : `user.registered`, `user.logged_in`.
- **Technologie** : mots de passe hachés avec Argon2id ; jetons JWT signés ; cookies `HttpOnly`.
- **Attention** : verrouillage temporaire après des échecs répétés, jamais de mot de passe dans les journaux. En phase 2, déléguer à un fournisseur OIDC (par exemple l'ENT de l'université).

#### 3. `users`

- **Rôle** : profil, langue, thème, rôle (`student`, `teacher`, `admin`), quotas individuels.
- **Données** : `users.profiles`, `users.roles`.
- **Interface** : `GET /users/me`, `PATCH /users/me`.
- **Dépendances** : `auth` pour l'identité.
- **Attention** : ne stocker que le nécessaire (pseudo, e-mail, langue) ; suppression de compte possible.

#### 4. `catalog`

- **Rôle** : catalogue versionné des cours, modules, leçons, exercices, mondes 3D et robots disponibles.
- **Données** : `courses`, `modules`, `lessons`, `exercises`, `exercise_versions`, `worlds`, `robots`.
- **Interface** : `GET /courses`, `/courses/{slug}`, `/lessons/{id}`, `/exercises/{id}`, `/worlds`, `/robots`.
- **Événements consommés** : aucun. **Émis** : `catalog.published`.
- **Attention** : une version publiée d'un exercice est immuable ; toute modification crée une nouvelle version, pour que les scores d'une tentative restent interprétables.

#### 5. `content`

- **Rôle** : fournit le texte Markdown des leçons, les images et le code de départ.
- **Données** : fichiers dans `content/` du dépôt, indexés par `catalog`.
- **Interface** : `GET /lessons/{id}/content?lang=fr`, `GET /lessons/{id}/starter`.
- **Attention** : le Markdown est nettoyé côté serveur (pas de HTML brut non filtré) ; les liens vers les images passent par une liste blanche de dossiers.

#### 6. `workspace`

- **Rôle** : conserve les fichiers de code de l'étudiant par leçon, afin qu'une nouvelle session reparte de son dernier travail.
- **Données** : `workspace_files` (utilisateur, leçon, chemin, contenu ou référence, version).
- **Interface** : `GET /workspaces/{lessonId}/files`, `PUT /workspaces/{lessonId}/files/{path}`.
- **Attention** : taille maximale par fichier et par espace ; à chaque démarrage de session, le contenu est copié dans le conteneur ; à l'arrêt, il est récupéré.

#### 7. `session-manager`

- **Rôle** : seul composant autorisé à parler à Docker. Crée un conteneur de session à partir de l'image `runtime`, lui applique ses limites, surveille son état, le supprime à l'arrêt ou à l'expiration.
- **Données** : `sessions`, `session_events`, `resource_quotas`.
- **Interface (interne)** : `POST /internal/sessions`, `DELETE /internal/sessions/{id}`, `GET /internal/sessions/{id}`.
- **Événements émis** : `session.created`, `session.running`, `session.failed`, `session.expired`, `session.stopped`.
- **Dépendances** : Docker Engine (socket local, jamais exposé), `events`, `workspace`.
- **Technologie** : Spring Boot avec docker-java, qui parle au démon Docker comme la commande `docker` ; tâche planifiée (@Scheduled) de réconciliation toutes les 30 secondes.
- **Attention** : c'est le composant le plus sensible ; il tourne sur un hôte dédié ou au moins avec un utilisateur restreint, et ses entrées sont validées strictement (aucun nom d'image libre, aucune option Docker fournie par l'utilisateur).

#### 8. `runtime`

- **Rôle** : l'image du conteneur de session. Contient ROS 2 Jazzy, PyBullet, le `sim_node`, rosbridge, un serveur de terminal et les outils de développement de l'étudiant.
- **Données** : fichiers de session dans un dossier temporaire du conteneur.
- **Interface** : rosbridge (WebSocket, port 9090), terminal (WebSocket, port 7681), contrôle de santé (`GET /healthz`, port 8081).
- **Attention** : image figée par numéro de version ; utilisateur non-root ; système de fichiers en lecture seule sauf `/home/student` et `/tmp` ; aucune connexion sortante sauf nécessité explicite.

#### 9. `realtime-proxy`

- **Rôle** : relie le navigateur au bon conteneur. Vérifie le jeton de session, route la connexion vers rosbridge ou le terminal, filtre les messages (liste blanche de topics), limite débit et taille des messages.
- **Données** : aucune persistante ; table de routage en mémoire alimentée par `session-manager`.
- **Interface** : `wss://…/ws/{sessionId}/ros` et `wss://…/ws/{sessionId}/terminal`.
- **Événements consommés** : `session.running`, `session.stopped`.
- **Technologie** : Spring Boot avec Spring WebSocket et la bibliothèque `Java-WebSocket` pour parler à rosbridge ; un seul processus suffit pour la démo.
- **Attention** : refuse toute opération ROS hors liste blanche (`call_service`, `advertise` sur des topics non autorisés) ; ferme la connexion quand la session expire.

#### 10. `progress`

- **Rôle** : enregistre les tentatives, les scores, l'avancement par leçon et par cours.
- **Données** : `attempts`, `scores`, `lesson_progress`, `hint_usage`.
- **Interface** : `POST /attempts`, `GET /attempts/{id}`, `GET /progress/me`.
- **Événements consommés** : `grading.completed`. **Émis** : `lesson.completed`, `course.completed`.
- **Attention** : une tentative est immuable ; la progression se recalcule à partir des tentatives.

#### 11. `grading`

- **Rôle** : exécute la vérification d'un exercice. Selon l'exercice : comparer la pose finale du robot à une cible, vérifier que `/cmd_vel` a été publié avec la bonne forme, comparer une carte à une carte de référence, ou contrôler une réponse à un questionnaire.
- **Données** : barèmes (`rubrics`), résultats (`grading_results`).
- **Interface (interne)** : tâche `grade_attempt(attemptId)` dans la file.
- **Événements** : consomme `attempt.submitted`, émet `grading.completed`.
- **Attention** : la vérification lit l'état de la simulation dans le conteneur (pose, topics enregistrés) mais n'exécute jamais de code étudiant en dehors de la session ; résultat déterministe pour une même entrée.

#### 12. `artifacts`

- **Rôle** : gère les fichiers volumineux : datasets (rosbag), cartes exportées, journaux de session, captures de trajectoire.
- **Données** : `datasets`, `dataset_versions`, `artifacts` (métadonnées SQL, binaire sur disque ou MinIO).
- **Interface** : `GET /datasets`, `GET /artifacts/{id}/download` (URL signée courte).
- **Attention** : chaque dataset porte source, licence, frames, unités et somme de contrôle ; rétention courte pour les artefacts de session.

#### 13. `events`

- **Rôle** : bus d'événements entre services et file des tâches asynchrones.
- **Technologie** : Redis Streams (suffisant pour la charge du projet) avec groupes de consommateurs ; Kafka n'est pas justifié.
- **Attention** : livraison au moins une fois ; chaque consommateur déduplique par `eventId` ; les événements ne contiennent aucune donnée personnelle inutile.

#### 14. `observability`

- **Rôle** : journaux structurés en JSON, métriques (sessions actives, durée de démarrage, CPU et mémoire par session, messages par seconde), alertes simples.
- **Technologie** : Prometheus + Grafana ; Loki si le temps le permet ; sinon `docker compose logs`.
- **Attention** : toujours inclure `request_id` et `session_id` dans les journaux ; ne jamais journaliser le code de l'étudiant ni les jetons.

#### 15. `admin` et `notifications` (phase 2)

- **Rôle** : `admin` liste les sessions actives, ajuste les quotas, désactive un compte ; `notifications` envoie des e-mails (confirmation d'inscription, fin de cours).
- **Attention** : hors MVP ; prévoir seulement les rôles et les tables dans le modèle de données pour ne pas avoir à migrer ensuite.

### 10.3 Contrats entre services

- **Appels synchrones** (HTTP JSON) pour les actions d'utilisateur : le front appelle `api` ; `api` appelle `session-manager` pour créer une session.
- **Événements asynchrones** (Redis Streams) pour tout ce qui peut attendre : vérification d'une tentative, mise à jour de la progression, nettoyage.
- **Pas de base partagée modifiable** : un seul PostgreSQL, mais un schéma par capacité, et chaque service n'écrit que dans son schéma.
- **Identifiants stables** : tous les identifiants sont des UUID ; les exercices, datasets et mondes sont référencés par une version immuable (`lidar-intro@1`).

## 11. Modèle de données PostgreSQL

Un seul serveur PostgreSQL, un schéma par capacité. Chaque module n'écrit que dans son schéma ; les autres modules le lisent par l'API interne, pas par des jointures directes, sauf pour les rapports en lecture seule.

### 11.1 Conventions

- Identifiants : `uuid` générés par l'application (UUID v7 si la bibliothèque le permet, pour garder l'ordre chronologique).
- Dates : `timestamptz` en UTC ; colonnes `created_at` et `updated_at` partout.
- Noms : `snake_case`, tables au pluriel, clés étrangères `<table>_id`.
- Suppression : suppression logique (`deleted_at`) pour les cours et les comptes ; physique pour les sessions expirées après la durée de rétention.
- Énumérations : types `enum` PostgreSQL ou `text` avec contrainte `CHECK` ; ne jamais stocker un entier sans signification.
- Migrations : Flyway, une migration par Merge Request, jamais de modification d'une migration déjà fusionnée.
- Données de départ : un script `seed` crée un cours d'exemple, 2 mondes, 1 robot et un compte de test.

### 11.2 Schémas et tables

| Schéma | Tables | Propriétaire |
| --- | --- | --- |
| `auth` | `credentials`, `refresh_tokens`, `login_attempts` | C |
| `users` | `profiles`, `roles`, `user_roles` | C |
| `catalog` | `courses`, `modules`, `lessons`, `exercises`, `exercise_versions`, `worlds`, `robots`, `rubrics` | C et D |
| `workspace` | `workspace_files` | C |
| `runtime` | `sessions`, `session_events`, `resource_quotas` | C |
| `progress` | `attempts`, `grading_results`, `lesson_progress`, `hint_usage` | C |
| `artifacts` | `datasets`, `dataset_versions`, `artifacts` | D |

### 11.3 Définition des tables principales

Les extraits suivants sont le point de départ des migrations ; les colonnes d'audit (`created_at`, `updated_at`) sont omises pour la lisibilité.

```sql
-- auth et users
CREATE TABLE auth.credentials (
  user_id        uuid PRIMARY KEY,
  email          citext UNIQUE NOT NULL,
  password_hash  text NOT NULL,            -- Argon2id
  failed_logins  int  NOT NULL DEFAULT 0,
  locked_until   timestamptz
);
CREATE TABLE auth.refresh_tokens (
  id           uuid PRIMARY KEY,
  user_id      uuid NOT NULL REFERENCES auth.credentials(user_id) ON DELETE CASCADE,
  token_hash   text NOT NULL,              -- on ne stocke jamais le jeton lui-même
  expires_at   timestamptz NOT NULL,
  revoked_at   timestamptz
);
CREATE TABLE users.profiles (
  user_id       uuid PRIMARY KEY,
  display_name  text NOT NULL,
  language      text NOT NULL DEFAULT 'fr' CHECK (language IN ('fr','en')),
  theme         text NOT NULL DEFAULT 'system',
  max_sessions  int  NOT NULL DEFAULT 1   -- quota individuel
);

-- catalog
CREATE TABLE catalog.courses (
  id          uuid PRIMARY KEY,
  slug        text UNIQUE NOT NULL,
  title_fr    text NOT NULL,
  title_en    text,
  level       text NOT NULL CHECK (level IN ('beginner','intermediate')),
  published   boolean NOT NULL DEFAULT false,
  deleted_at  timestamptz
);
CREATE TABLE catalog.lessons (
  id          uuid PRIMARY KEY,
  course_id   uuid NOT NULL REFERENCES catalog.courses(id),
  module_id   uuid REFERENCES catalog.modules(id),
  position    int  NOT NULL,
  slug        text NOT NULL,
  content_ref text NOT NULL,               -- chemin dans content/
  UNIQUE (course_id, slug)
);
CREATE TABLE catalog.exercise_versions (
  id            uuid PRIMARY KEY,
  exercise_key  text NOT NULL,             -- ex. 'lidar-intro'
  version       int  NOT NULL,
  lesson_id     uuid NOT NULL REFERENCES catalog.lessons(id),
  world_id      uuid NOT NULL REFERENCES catalog.worlds(id),
  robot_id      uuid NOT NULL REFERENCES catalog.robots(id),
  config        jsonb NOT NULL,            -- paramètres du simulateur
  rubric_id     uuid NOT NULL REFERENCES catalog.rubrics(id),
  UNIQUE (exercise_key, version)
);
CREATE TABLE catalog.worlds (
  id        uuid PRIMARY KEY,
  key       text UNIQUE NOT NULL,          -- ex. 'simple-room'
  file_ref  text NOT NULL,                 -- JSON du monde
  checksum  text NOT NULL
);
CREATE TABLE catalog.robots (
  id        uuid PRIMARY KEY,
  key       text UNIQUE NOT NULL,          -- ex. 'diffbot-v1'
  urdf_ref  text NOT NULL,
  checksum  text NOT NULL
);

-- runtime
CREATE TYPE runtime.session_state AS ENUM
  ('requested','starting','running','stopping','stopped','expired','failed');
CREATE TABLE runtime.sessions (
  id                  uuid PRIMARY KEY,
  user_id             uuid NOT NULL,
  exercise_version_id uuid REFERENCES catalog.exercise_versions(id),
  world_id            uuid NOT NULL,
  robot_id            uuid NOT NULL,
  state               runtime.session_state NOT NULL,
  container_id        text,                -- géré par session-manager uniquement
  ros_domain_id       int,
  started_at          timestamptz,
  expires_at          timestamptz NOT NULL,
  stopped_reason      text,
  idempotency_key     text,
  UNIQUE (user_id, idempotency_key)
);
CREATE INDEX ON runtime.sessions (user_id, state);
CREATE INDEX ON runtime.sessions (state, expires_at);
CREATE TABLE runtime.session_events (
  id          bigserial PRIMARY KEY,
  session_id  uuid NOT NULL REFERENCES runtime.sessions(id) ON DELETE CASCADE,
  type        text NOT NULL,
  payload     jsonb,
  at          timestamptz NOT NULL DEFAULT now()
);

-- progress
CREATE TABLE progress.attempts (
  id                  uuid PRIMARY KEY,
  user_id             uuid NOT NULL,
  session_id          uuid REFERENCES runtime.sessions(id) ON DELETE SET NULL,
  exercise_version_id uuid NOT NULL REFERENCES catalog.exercise_versions(id),
  answers             jsonb,
  submitted_at        timestamptz NOT NULL DEFAULT now(),
  status              text NOT NULL DEFAULT 'pending'
                      CHECK (status IN ('pending','graded','error'))
);
CREATE TABLE progress.grading_results (
  attempt_id     uuid PRIMARY KEY REFERENCES progress.attempts(id),
  score          numeric(5,2) NOT NULL,
  max_score      numeric(5,2) NOT NULL,
  details        jsonb NOT NULL,           -- un élément par critère du barème
  rubric_version int NOT NULL,
  graded_at      timestamptz NOT NULL DEFAULT now()
);
CREATE TABLE progress.lesson_progress (
  user_id     uuid NOT NULL,
  lesson_id   uuid NOT NULL REFERENCES catalog.lessons(id),
  best_score  numeric(5,2),
  completed   boolean NOT NULL DEFAULT false,
  last_opened timestamptz,
  PRIMARY KEY (user_id, lesson_id)
);

-- workspace et artifacts
CREATE TABLE workspace.workspace_files (
  user_id    uuid NOT NULL,
  lesson_id  uuid NOT NULL,
  path       text NOT NULL,
  content    text NOT NULL,
  version    int  NOT NULL DEFAULT 1,
  PRIMARY KEY (user_id, lesson_id, path)
);
CREATE TABLE artifacts.dataset_versions (
  id          uuid PRIMARY KEY,
  key         text NOT NULL,
  version     int  NOT NULL,
  format      text NOT NULL,               -- 'rosbag2', 'json', 'pcd'
  source      text NOT NULL,               -- provenance
  license     text NOT NULL,
  frame_ids   text[] NOT NULL,
  checksum    text NOT NULL,
  object_ref  text NOT NULL,
  UNIQUE (key, version)
);
```

### 11.4 Règles d'intégrité et de rétention

- **Immuabilité** : une ligne de `exercise_versions`, `dataset_versions` et `attempts` ne se modifie pas après publication ; un correctif crée une nouvelle version.
- **Un seul score par tentative**, plusieurs tentatives par exercice ; `lesson_progress.best_score` est recalculé par un événement, pas à la main.
- **Quotas** : au plus `max_sessions` sessions `starting` ou `running` par utilisateur, vérifié dans la transaction qui crée la session (verrou sur la ligne de l'utilisateur).
- **Rétention** : sessions `stopped` ou `expired` supprimées après 30 jours ; `session_events` après 14 jours ; artefacts de session après 7 jours ; tentatives et scores conservés tant que le compte existe.
- **Suppression d'un compte** : suppression des identifiants, du profil et de l'espace de travail ; les tentatives sont anonymisées (`user_id` remplacé par un identifiant aléatoire).
- **Sauvegardes** : export `pg_dump` quotidien du serveur de démo, copié hors de la machine, restauration testée une fois avant la démo finale.

## 12. Spécification de l'API et des événements

Ce chapitre fixe les contrats entre le front (B), l'API (C), le gestionnaire de sessions (C), le simulateur (A) et les vérifications (A et D). Toute modification passe par une Merge Request qui touche `docs/contracts.md` et le schéma OpenAPI.

### 12.1 Conventions générales

| Sujet | Règle |
| --- | --- |
| Adresse de base | `https://<hôte>/api/v1` |
| Format | JSON UTF-8 ; dates ISO 8601 en UTC ; identifiants UUID |
| Authentification | Cookie `access_token` (`HttpOnly`) ; l'en-tête `Authorization: Bearer` est accepté pour Postman et les tests |
| Schéma | OpenAPI 3 généré par springdoc-openapi, publié sur `/api/v1/openapi.json`, documenté sur `/api/docs` en développement |
| Pagination | `?limit=20&cursor=<opaque>` ; réponse `{ items: [], nextCursor: string \| null }` |
| Idempotence | En-tête `Idempotency-Key` sur `POST /sessions` et `POST /attempts` ; même clé = même résultat |
| Limitation de débit | 60 requêtes par minute par utilisateur ; 10 par minute sur `/auth/*` ; réponse `429` avec `Retry-After` |
| Versions | Préfixe `/v1` ; un changement incompatible crée `/v2` |
| Correlation | En-tête `X-Request-Id` accepté ou généré, renvoyé dans la réponse et dans les journaux |

**Format d'erreur unique**

```json
{
  "error": {
    "code": "session_quota_exceeded",
    "message": "Vous avez déjà une session en cours.",
    "requestId": "b4c1…",
    "details": { "maxSessions": 1 }
  }
}
```

| Statut | Quand | Codes machine (exemples) |
| --- | --- | --- |
| 400 | Corps ou paramètres invalides | `validation_error` |
| 401 | Non connecté ou jeton expiré | `unauthenticated`, `token_expired` |
| 403 | Connecté mais sans droit | `forbidden`, `not_session_owner` |
| 404 | Ressource inconnue (ou non visible) | `not_found` |
| 409 | Conflit d'état | `session_not_running`, `already_submitted` |
| 422 | Valeur correcte mais refusée par une règle | `exercise_not_published` |
| 429 | Trop de requêtes ou quota | `rate_limited`, `session_quota_exceeded` |
| 503 | Capacité de simulation pleine | `capacity_full` (avec `Retry-After`) |

### 12.2 Authentification et profil

| Méthode et chemin | Rôle | Corps ou paramètres | Réponse |
| --- | --- | --- | --- |
| `POST /auth/register` | Créer un compte | `{ email, password, displayName }` | `201` + profil ; cookies posés |
| `POST /auth/login` | Se connecter | `{ email, password }` | `200` + profil ; cookies posés |
| `POST /auth/refresh` | Renouveler l'accès | cookie de rafraîchissement | `204` ; nouveau cookie d'accès |
| `POST /auth/logout` | Se déconnecter | — | `204` ; jeton révoqué |
| `GET /users/me` | Lire son profil | — | `{ id, email, displayName, language, theme, role }` |
| `PATCH /users/me` | Modifier son profil | champs partiels | profil mis à jour |
| `DELETE /users/me` | Supprimer son compte | confirmation du mot de passe | `204` |

### 12.3 Catalogue et contenu

| Méthode et chemin | Rôle | Réponse (résumé) |
| --- | --- | --- |
| `GET /courses?level=&q=` | Lister les cours publiés | `{ items: [{ id, slug, title, level, lessonCount }] }` |
| `GET /courses/{slug}` | Fiche d'un cours | `{ id, title, description, prerequisites, modules: [{ title, lessons: [{ id, slug, title, estimatedMinutes }] }] }` |
| `GET /lessons/{id}` | Métadonnées de la leçon | `{ id, title, objectives, exerciseVersionId, worldKey, robotKey }` |
| `GET /lessons/{id}/content?lang=fr` | Texte Markdown | `{ markdown, hints: [{ id, text }] }` (les indices sont envoyés un par un) |
| `GET /lessons/{id}/starter` | Code de départ | `{ files: [{ path, content }] }` |
| `GET /worlds` | Mondes disponibles | `{ items: [{ key, title, previewUrl }] }` |
| `GET /worlds/{key}` | JSON du monde pour Three.js | document de monde (section 13.4) |
| `GET /robots/{key}/urdf` | URDF du robot | fichier XML |

### 12.4 Sessions

| Méthode et chemin | Rôle | Corps | Réponse |
| --- | --- | --- | --- |
| `POST /sessions` | Demander une session | voir ci-dessous | `202` + session en état `requested` |
| `GET /sessions/{id}` | État de la session | — | session complète |
| `GET /sessions?state=active` | Mes sessions actives | — | liste |
| `POST /sessions/{id}/extend` | Prolonger | — | session avec nouvelle expiration, ou `409` |
| `DELETE /sessions/{id}` | Arrêter | — | `202` ; idempotent (une session déjà arrêtée renvoie `200`) |

**Création d'une session**

```json
POST /api/v1/sessions
Idempotency-Key: 6f1a…
{
  "lessonId": "0b5c…",
  "mode": "lesson",
  "worldKey": null,
  "robotKey": null
}
```

```json
202 Accepted
{
  "id": "c9d2…",
  "state": "requested",
  "expiresAt": "2026-10-13T14:30:00Z",
  "connection": null
}
```

Quand l'état devient `running`, `GET /sessions/{id}` renvoie le bloc de connexion :

```json
{
  "id": "c9d2…",
  "state": "running",
  "expiresAt": "2026-10-13T14:30:00Z",
  "connection": {
    "rosbridgeUrl": "wss://exemple.fr/ws/c9d2…/ros",
    "terminalUrl": "wss://exemple.fr/ws/c9d2…/terminal",
    "token": "<jeton court, valable pour cette session>"
  },
  "world": "simple-room",
  "robot": "diffbot-v1"
}
```

Pour un `mode` égal à `playground`, `worldKey` et `robotKey` sont obligatoires et `lessonId` est nul.

### 12.5 Espace de travail, tentatives et progression

| Méthode et chemin | Rôle | Remarque |
| --- | --- | --- |
| `GET /workspaces/{lessonId}/files` | Lister les fichiers sauvegardés | Créés depuis le code de départ au premier accès |
| `PUT /workspaces/{lessonId}/files/{path}` | Écrire un fichier | Taille maximale 200 Ko ; chemin relatif sans `..` |
| `DELETE /workspaces/{lessonId}/files/{path}` | Supprimer un fichier | — |
| `POST /attempts` | Soumettre | `{ sessionId, answers }` ; `202` ; le score arrive de façon asynchrone |
| `GET /attempts/{id}` | Résultat d'une tentative | `{ status, score, maxScore, criteria: [{ key, passed, points, feedback }] }` |
| `GET /attempts?lessonId=` | Historique | paginé |
| `GET /progress/me` | Vue d'ensemble | `{ courses: [{ slug, completedLessons, totalLessons, bestScore }] }` |

### 12.6 Événements (Redis Streams)

Chaque événement a une enveloppe commune : `{ eventId, type, occurredAt, producer, payload }`. Les consommateurs dédupliquent par `eventId`.

| Événement | Producteur | Consommateurs | Contenu du `payload` |
| --- | --- | --- | --- |
| `session.requested` | `api` | `session-manager` | `sessionId`, `userId`, `exerciseVersionId`, `worldKey`, `robotKey` |
| `session.running` | `session-manager` | `api`, `realtime-proxy` | `sessionId`, `containerAddress`, `rosbridgePort`, `terminalPort` |
| `session.failed` | `session-manager` | `api` | `sessionId`, `reason` |
| `session.stopped` | `session-manager` | `api`, `realtime-proxy`, `workspace` | `sessionId`, `reason` (`user`, `expired`, `error`) |
| `attempt.submitted` | `api` | `grading` | `attemptId`, `sessionId`, `exerciseVersionId` |
| `grading.completed` | `grading` | `api` (`progress`) | `attemptId`, `score`, `maxScore`, `criteria` |
| `lesson.completed` | `progress` | `notifications` (phase 2) | `userId`, `lessonId`, `score` |
| `catalog.published` | `catalog` | `content` | `courseId`, `version` |

### 12.7 API interne du gestionnaire de sessions

Appelée uniquement par `api`, sur le réseau interne, avec un secret partagé en en-tête.

| Méthode et chemin | Rôle |
| --- | --- |
| `POST /internal/sessions` | Créer un conteneur : `{ sessionId, image, worldKey, robotKey, files[], limits }` |
| `GET /internal/sessions/{id}` | État du conteneur (créé, démarré, sain, arrêté) |
| `DELETE /internal/sessions/{id}` | Arrêter et supprimer ; récupérer les fichiers de l'étudiant |
| `GET /internal/capacity` | Sessions actives, maximum, mémoire et CPU restants |

Le champ `image` est choisi parmi une liste blanche de l'environnement ; il n'est jamais copié depuis la requête d'un utilisateur.

### 12.8 Protocole WebSocket

**ROS (`/ws/{sessionId}/ros`)** : le protocole est celui de rosbridge (JSON). Le proxy n'accepte que :

| Opération rosbridge | Autorisée | Condition |
| --- | --- | --- |
| `subscribe` / `unsubscribe` | Oui | Topic et type dans la liste blanche de l'exercice ; cadence maximale imposée (`throttle_rate`) |
| `publish` | Oui | Seulement `/cmd_vel` (et les topics listés par l'exercice) ; fréquence maximale 30 messages par seconde ; vitesse bornée |
| `advertise` | Oui | Seulement pour les topics publiables |
| `call_service` | Non par défaut | Autorisé service par service dans l'exercice |
| `set_level`, `fragment`, autres | Non | — |

**Terminal (`/ws/{sessionId}/terminal`)** : messages JSON `{ type: 'input', data }`, `{ type: 'resize', cols, rows }` du client ; `{ type: 'output', data }`, `{ type: 'exit', code }` du serveur. Taille maximale d'un message : 64 Ko.

**Authentification** : le jeton court est envoyé dans le premier message `{ type: 'auth', token }` ; sans message valide dans les 5 secondes, la connexion est fermée (code 4401). Le proxy ferme toutes les connexions d'une session à l'événement `session.stopped`.

## 13. Simulateur : conception détaillée

Ce chapitre est la référence de A. Il décrit le package ROS 2, les nodes, les formats de fichiers, les algorithmes et ce que le simulateur ne fait pas.

### 13.1 Packages ROS 2

| Package | Type | Contenu |
| --- | --- | --- |
| `robot_edu_description` | `ament_python` ou données seules | URDF des robots, maillages simples, paramètres |
| `robot_edu_worlds` | données | Fichiers JSON des mondes et schéma JSON |
| `robot_edu_sim` | `ament_python` | `sim_node`, modèles physiques, LiDAR, bruit |
| `robot_edu_bringup` | `ament_python` | Fichiers `launch`, paramètres par exercice |
| `robot_edu_eval` | `ament_python` | Outils de vérification utilisés par `grading` (pose finale, enregistrement de topics) |
| `robot_edu_examples` | `ament_python` | Corrigés et exemples de code pour les leçons |

### 13.2 Le node `sim_node`

**Boucle principale** (cadence fixe, horloge simulée) :

1. Lire la dernière commande `/cmd_vel` reçue ; si elle date de plus de 0,5 s de temps simulé, la considérer comme nulle (sécurité comme sur un vrai robot).
2. Convertir la commande en vitesses de roues et les appliquer aux articulations avec `setJointMotorControl2` en mode vitesse.
3. Avancer la physique de `n` pas de 1/240 s pour couvrir l'intervalle de la boucle (par exemple 8 pas pour 30 Hz).
4. Lire la pose de la base, l'orientation et les vitesses ; construire `/odom` et le transform `odom` → `base_link`.
5. À chaque période du LiDAR, lancer les rayons et publier `/scan` (et `/points` si activé).
6. Publier `/joint_states`, `/tf` et `/clock`.

**Paramètres du node** (fichier YAML, un par exercice)

| Paramètre | Exemple | Rôle |
| --- | --- | --- |
| `world_file` | `simple-room.json` | Monde à charger |
| `robot_urdf` | `diffbot-v1.urdf` | Robot à charger |
| `physics_hz` | `240` | Fréquence de la physique |
| `loop_hz` | `30` | Fréquence de publication de l'odométrie |
| `lidar.rate_hz` | `10` | Fréquence du scan |
| `lidar.num_rays` | `360` | Nombre de rayons |
| `lidar.range_min`, `lidar.range_max` | `0.12`, `8.0` | Portées en mètres |
| `lidar.noise_std` | `0.0` | Écart-type du bruit gaussien (m) ; > 0 pour les exercices de SLAM |
| `odom.noise` | `false` | Ajoute une dérive à l'odométrie |
| `seed` | `42` | Graine aléatoire pour la reproductibilité |
| `real_time_factor` | `1.0` | Vitesse de la simulation par rapport à l'horloge réelle |

### 13.3 Cinématique du robot différentiel

Pour une commande `v` (vitesse linéaire, m/s) et `ω` (vitesse angulaire, rad/s), avec un rayon de roue `r` et un entraxe `L` :

```latex
\omega_{gauche} = \frac{v - \omega L / 2}{r} \qquad \omega_{droite} = \frac{v + \omega L / 2}{r}
```

L'odométrie publiée vient de l'état physique réel de la base (avec dérive optionnelle), pas d'une intégration parfaite de la commande ; c'est ce qui rend l'écart entre odométrie et vérité terrain observable dans les leçons. Les vitesses sont plafonnées (par exemple 0,5 m/s et 1,5 rad/s) pour éviter des scènes instables.

### 13.4 Format du fichier de monde

Un seul fichier JSON décrit le monde pour PyBullet et pour Three.js. Un schéma JSON versionné (`world.schema.json`) le valide dans la CI.

```json
{
  "schemaVersion": 1,
  "key": "simple-room",
  "title": "Pièce simple",
  "units": "meters",
  "bounds": { "x": [-5, 5], "y": [-5, 5] },
  "spawn": { "x": 0.0, "y": 0.0, "yaw": 0.0 },
  "floor": { "friction": 0.8, "color": "#d9d9d9" },
  "walls": [
    { "id": "w-north", "from": [-4, 4], "to": [4, 4], "height": 1.0, "thickness": 0.1 },
    { "id": "w-east",  "from": [4, 4],  "to": [4, -4], "height": 1.0, "thickness": 0.1 }
  ],
  "boxes": [
    { "id": "obs-1", "center": [1.5, 0.5], "size": [0.5, 0.5, 0.5], "yaw": 0.3 }
  ],
  "cylinders": [
    { "id": "pole-1", "center": [-2, 1], "radius": 0.15, "height": 1.0 }
  ],
  "goals": [
    { "id": "target", "center": [3, -2], "radius": 0.3 }
  ]
}
```

- Les murs, boîtes et cylindres sont les seules primitives du MVP : simples à lancer en rayons et à afficher.
- Les objectifs (`goals`) servent à `grading` (le robot est-il arrivé dans le cercle ?) et sont affichés dans la vue 3D.
- Un `id` stable permet à la vue 3D et au simulateur de parler du même objet.

### 13.5 Description du robot (URDF)

Le robot `diffbot-v1` est un robot différentiel simple :

| Lien | Forme | Remarque |
| --- | --- | --- |
| `base_link` | boîte 0,30 × 0,20 × 0,10 m | masse 3 kg |
| `left_wheel`, `right_wheel` | cylindres, rayon 0,05 m | articulations `continuous`, moteur en vitesse |
| `caster` | sphère sans frottement | roulette folle à l'arrière |
| `laser` | petit cylindre, 0,10 m au-dessus de la base | repère du capteur ; à z = 0,15 m |
| `camera_link` (optionnel) | boîte | pour une caméra simulée en phase 2 |

Le même URDF est servi au navigateur (`GET /robots/{key}/urdf`) ; les maillages sont des primitives pour le MVP, ce qui évite les problèmes de chemins `package://` dans `urdf-loader`.

### 13.6 LiDAR par lancer de rayons

```python
import numpy as np, pybullet as p

def scan(sensor_pos, sensor_yaw, n, rmin, rmax, noise_std, rng):
    angles = sensor_yaw + np.linspace(-np.pi, np.pi, n, endpoint=False)
    starts = [sensor_pos] * n
    ends = [[sensor_pos[0] + rmax * np.cos(a),
             sensor_pos[1] + rmax * np.sin(a),
             sensor_pos[2]] for a in angles]
    hits = p.rayTestBatch(starts, ends)
    ranges = []
    for h in hits:
        frac = h[2]                      # fraction du segment avant l'impact
        r = frac * rmax if h[0] != -1 else float('inf')
        if r < rmin: r = float('inf')
        if np.isfinite(r) and noise_std > 0:
            r += rng.normal(0.0, noise_std)
        ranges.append(r)
    return ranges
```

- Le message `LaserScan` est rempli avec `angle_min = -π`, `angle_increment = 2π / n`, `range_min`, `range_max` et `ranges`.
- Un impact absent ou hors portée donne `inf`, comme sur un vrai LiDAR.
- Un LiDAR 3D est obtenu en répétant le lancer sur plusieurs plans d'élévation (16 plans par exemple) et en publiant un `PointCloud2` ; il coûte 16 fois plus de rayons, donc il reste optionnel et à faible cadence.
- Les rayons sont lancés en un seul appel par scan (`rayTestBatch`) pour limiter le coût Python.

### 13.7 Temps simulé, TF et frames

| Chaîne | Publiée par | Remarque |
| --- | --- | --- |
| `map` → `odom` | Le SLAM de l'étudiant (ou transform statique identité avant le SLAM) | Le simulateur publie l'identité tant qu'aucun SLAM n'est lancé |
| `odom` → `base_link` | `sim_node` | Issue de l'état de la base |
| `base_link` → `laser` | `robot_state_publisher` ou `sim_node` | Transform fixe issu de l'URDF |
| `base_link` → `left_wheel`, `right_wheel` | `robot_state_publisher` | D'après `/joint_states` |

Tous les nodes de l'étudiant utilisent `use_sim_time:=true`. L'horloge `/clock` est publiée à 50 Hz par `sim_node`. Un horodatage dans le futur ou dans le passé lointain est une erreur fréquente ; la leçon sur les frames l'illustre volontairement.

### 13.8 Déterminisme et performance

- **Déterminisme** : pas de physique fixe, graine aléatoire dans les paramètres, pas de dépendance à l'horloge murale. Deux exécutions avec les mêmes commandes donnent la même trajectoire à la tolérance numérique près ; c'est ce qui rend la vérification fiable.
- **Budget de calcul** : viser moins de 25 % d'un cœur CPU par session pour un scan de 360 rayons à 10 Hz et une physique à 240 Hz. À mesurer dès la semaine 3, car c'est ce qui détermine le nombre de sessions simultanées.
- **Mesures** : le node publie sur `/sim/stats` (temps moyen d'une boucle, facteur temps réel) pour détecter un conteneur surchargé.

### 13.9 Limites à afficher dans les leçons

- Pas de dynamique fine des pneus ni de glissement réaliste ; les roues adhèrent bien au sol.
- LiDAR parfait, sans réflexion spéculaire ni verre ; bruit gaussien seulement.
- Pas de capteurs optiques réalistes (la caméra, si elle existe, est un rendu simplifié).
- Les résultats ne valident pas un robot réel ; ils servent à comprendre les interfaces et les algorithmes.

## 14. Frontend : structure du code, composants et exemples

Référence de B. Le front est une application React en TypeScript. Elle sert les pages du chapitre 9 et relie le navigateur à l'API (HTTP) et au proxy temps réel (WebSocket).

### 14.1 Pile technique

| Besoin | Choix | Pourquoi |
| --- | --- | --- |
| Framework | React 18 + TypeScript | Large écosystème, bon support de Three.js et de Monaco |
| Outil de build | Vite | Démarrage rapide, découpage automatique du code |
| Routage | React Router | Routes du chapitre 9, routes protégées |
| Données serveur | TanStack Query | Cache, relances, états de chargement pour les appels HTTP |
| État local de la session | Zustand | Petit, adapté aux flux de données temps réel |
| 3D | Three.js + `urdf-loader` | Rendu du monde et du robot URDF |
| Lien ROS | `roslib` (rosbridge) | Client WebSocket mature pour les topics |
| Éditeur | `@monaco-editor/react` | Même éditeur que VS Code |
| Terminal | `@xterm/xterm` + addon `fit` | Terminal dans le navigateur |
| Texte des leçons | `react-markdown` + `rehype-sanitize` | Markdown sûr (pas de HTML non filtré) |
| Langues | `i18next` | Français et anglais |
| Tests | Vitest, Testing Library, Playwright | Unitaires, composants, bout en bout |
| Qualité | ESLint, Prettier, `tsc --noEmit` | Vérifiés en CI |

### 14.2 Arborescence du dossier `web/`

```text
web/
  index.html
  package.json
  vite.config.ts
  tsconfig.json
  public/
    favicon.svg
    locales/fr/common.json   locales/en/common.json
  src/
    main.tsx                 point d'entrée
    app/
      App.tsx                routes
      routes.tsx             définition des routes et gardes
      providers.tsx          Query, i18n, thème
    shared/
      api/client.ts          fetch avec cookies, erreurs, X-Request-Id
      api/types.ts           types générés depuis OpenAPI
      ui/                    Button, Tabs, Panel, Modal, Toast, Spinner
      styles/tokens.css      couleurs, espacements, thèmes
      hooks/                 useMediaQuery, useInterval…
    features/
      auth/                  LoginPage, RegisterPage, useAuth
      courses/               CoursesPage, CoursePage, LessonList
      lesson/
        LessonPage.tsx       assemble les 4 panneaux
        LessonText.tsx       Markdown + indices
        SessionBar.tsx       état, minuteur, arrêt
        SubmitPanel.tsx
      session/
        useSession.ts        création, interrogation, arrêt
        sessionStore.ts      état de connexion et de session
      editor/
        CodeEditor.tsx       Monaco, enregistrement automatique
        useWorkspace.ts
      terminal/
        TerminalPanel.tsx    xterm + WebSocket
        terminalClient.ts
      viewer3d/
        Viewer3D.tsx         conteneur React de la scène
        scene/SceneManager.ts    renderer, caméras, boucle de rendu
        scene/frames.ts      conversion ROS vers Three.js (un seul endroit)
        layers/WorldLayer.ts RobotModel.ts ScanLayer.ts
        layers/TrailLayer.ts MapLayer.ts PointCloudLayer.ts
        ui/LayerToggles.tsx CameraControls.tsx SourceBadge.tsx
      ros/
        rosClient.ts         connexion rosbridge, abonnements
        topicStore.ts        dernière valeur de chaque topic
        messages.ts          types TypeScript des messages ROS
      progress/              DashboardPage, ProgressPage
      profile/               ProfilePage
    i18n/index.ts
  tests/
    unit/  e2e/
```

### 14.3 Flux de données temps réel

1. `rosClient` ouvre le WebSocket avec le jeton de session et s'abonne aux topics de la leçon.
2. Chaque message est validé (champs attendus, nombres finis) puis stocké dans `topicStore` : seule la **dernière valeur** de chaque topic est gardée.
3. La boucle de rendu Three.js lit `topicStore` à chaque image ; elle ne se déclenche pas à chaque message.
4. Les composants React qui affichent des valeurs (barre basse, cadence des topics) lisent le même store à faible fréquence (4 Hz).

Ce découplage évite que 10 scans par seconde provoquent 10 rendus React par seconde.

### 14.4 Conversion des repères (module unique)

```ts
// scene/frames.ts — à utiliser partout, jamais de conversion ailleurs
import { Vector3, Quaternion } from 'three';

// ROS : x avant, y gauche, z haut.   Three.js : x droite, y haut, z vers la caméra.
// Choix du projet : x_three = x_ros, y_three = z_ros, z_three = -y_ros.
export function rosToThreePosition(x: number, y: number, z: number): Vector3 {
  return new Vector3(x, z, -y);
}

export function rosToThreeQuaternion(q: { x: number; y: number; z: number; w: number }): Quaternion {
  // Même changement de base appliqué à la partie vectorielle du quaternion
  return new Quaternion(q.x, q.z, -q.y, q.w);
}
```

### 14.5 Client rosbridge

```ts
// ros/rosClient.ts (extrait)
import ROSLIB from 'roslib';
import { useTopicStore } from './topicStore';

export class RosClient {
  private ros: ROSLIB.Ros;
  private subs = new Map<string, ROSLIB.Topic>();

  constructor(private url: string, private token: string) {
    this.ros = new ROSLIB.Ros({ url });
    this.ros.on('connection', () => this.send({ type: 'auth', token }));
    this.ros.on('close', () => this.scheduleReconnect());
  }

  subscribe(name: string, messageType: string, throttleMs = 100) {
    const topic = new ROSLIB.Topic({
      ros: this.ros, name, messageType, throttle_rate: throttleMs, queue_length: 1,
    });
    topic.subscribe((msg) => {
      if (isValid(name, msg)) useTopicStore.getState().set(name, msg, performance.now());
    });
    this.subs.set(name, topic);
  }

  publishCmdVel(linear: number, angular: number) {
    const topic = new ROSLIB.Topic({ ros: this.ros, name: '/cmd_vel', messageType: 'geometry_msgs/Twist' });
    topic.publish({ linear: { x: linear, y: 0, z: 0 }, angular: { x: 0, y: 0, z: angular } });
  }

  close() { this.subs.forEach((t) => t.unsubscribe()); this.ros.close(); }
  // send, scheduleReconnect (délai 1 s, 2 s, 4 s, max 15 s), isValid : voir le dépôt
}
```

### 14.6 Couche de scan

```ts
// viewer3d/layers/ScanLayer.ts (extrait)
import { BufferGeometry, Float32BufferAttribute, Points, PointsMaterial } from 'three';
import { rosToThreePosition } from '../scene/frames';

export class ScanLayer {
  readonly object: Points;
  private positions: Float32Array;
  constructor(maxPoints = 2048) {
    this.positions = new Float32Array(maxPoints * 3);
    const g = new BufferGeometry();
    g.setAttribute('position', new Float32BufferAttribute(this.positions, 3));
    this.object = new Points(g, new PointsMaterial({ size: 0.04 }));
  }

  update(scan: LaserScan, laserPoseInWorld: (x: number, y: number) => [number, number, number]) {
    let n = 0;
    for (let i = 0; i < scan.ranges.length; i++) {
      const r = scan.ranges[i];
      if (!Number.isFinite(r) || r < scan.range_min || r > scan.range_max) continue;
      const a = scan.angle_min + i * scan.angle_increment;
      const [wx, wy, wz] = laserPoseInWorld(r * Math.cos(a), r * Math.sin(a));
      const v = rosToThreePosition(wx, wy, wz);
      this.positions.set([v.x, v.y, v.z], 3 * n++);
    }
    const geom = this.object.geometry;
    (geom.attributes.position as Float32BufferAttribute).needsUpdate = true;
    geom.setDrawRange(0, n);
  }
}
```

Le tampon est alloué une seule fois et réutilisé ; on ne crée pas de nouvelle géométrie à chaque scan.

### 14.7 Machine d'états de la session côté front

| État du front | Déclencheur | Interface |
| --- | --- | --- |
| `idle` | Aucune session | Bouton « Démarrer la simulation » |
| `requesting` | Clic sur Démarrer | Envoi de `POST /sessions` |
| `starting` | Réponse `202` | Écran de progression, interrogation toutes les 1,5 s |
| `connecting` | Session `running` | Ouverture des deux WebSockets |
| `live` | WebSockets authentifiés | Page de leçon active |
| `reconnecting` | Coupure réseau | Bandeau, essais avec délai croissant |
| `ended` | Arrêt, expiration ou échec | Fenêtre de fin ; code sauvegardé |
| `error` | Échec de création ou de connexion | Message et bouton Réessayer |

### 14.8 Bonnes pratiques de performance

- Charger Three.js, Monaco et xterm.js en import dynamique, uniquement sur la page de leçon.
- Limiter le rendu à l'affichage visible (`document.visibilityState`) ; mettre en pause quand l'onglet est caché.
- Réutiliser géométries et matériaux, libérer (`dispose`) les ressources en quittant la page.
- Pour les nuages de points volumineux, limiter le nombre de points affichés (décimation) et la cadence à 5 Hz.
- Mesurer avec les outils de performance du navigateur : viser 60 images par seconde avec un scan de 360 points et une trajectoire de 5 000 points.

## 15. Plan du dépôt

Le projet vit dans un **monorepo GitLab** : un seul dépôt contient le front, les services, l'image du simulateur, le contenu des cours, l'infrastructure et la documentation. Avec 4 personnes et un contrat d'interface qui bouge souvent au début, un seul dépôt évite les versions désynchronisées et permet une Merge Request qui touche l'API et le front en même temps.

### 15.1 Arborescence complète

```text
robot-edu/
  README.md                    présentation, démarrage en 5 commandes
  CONTRIBUTING.md             règles de branches, commits, revues
  LICENSE
  Makefile                    raccourcis : make up, make test, make seed…
  compose.yaml                services communs (db, redis, api, proxy, web)
  compose.dev.yaml            surcharges de développement
  compose.prod.yaml           surcharges de démonstration (limites, restart)
  .env.example                variables sans valeurs secrètes
  .gitignore  .editorconfig  .pre-commit-config.yaml
  .gitlab-ci.yml              pipeline CI
  .gitlab/
    CODEOWNERS                propriétaires par dossier
    issue_templates/          bug.md  feature.md  task.md
    merge_request_templates/  default.md

  docs/
    architecture/             ce document et ses schémas
    uml/                      cas d'utilisation, classes, séquence, activité (sources + PNG/PDF)
    contracts.md              interfaces figées (topics, API, événements)
    frames.md                 repères et conversions
    adr/                      décisions d'architecture, une par fichier
    api/openapi.json          généré par springdoc, ne pas éditer à la main
    exercises/                fiche de vérification par exercice
    acceptance/               scénarios de recette
    runbooks/                 démarrer, sauvegarder, dépanner
    postman/                  collection et environnements (sans secrets)
    rendus/                   CdC initial, CdC final, documentation technique, guide d'utilisation

  services/                   projet Maven multi-modules (pom.xml parent)
    pom.xml
    api/                      application Spring Boot principale
      pom.xml  Dockerfile
      src/main/java/com/robotedu/api/
        RobotEduApplication.java
        model/                entités JPA : User, Role, Course, Lesson, Exercise,
                              Enrollment, Session, Attempt, Score, WorkspaceFile
        repository/           interfaces JpaRepository
        service/              logique métier : AuthService, CatalogService, SessionService,
                              ProgressService, GradingService, WorkspaceService
        controller/           contrôleurs REST (@RestController)
        dto/                  objets d'échange validés par Bean Validation
        security/             JwtAuthFilter, SecurityConfig
        config/               CORS, OpenAPI, Redis, propriétés
        exception/            ApiException, @RestControllerAdvice
        event/                publication et consommation des événements (Redis Streams)
      src/main/resources/
        application.yml  application-dev.yml
        db/migration/         Flyway : V1__init.sql, V2__…
      src/test/java/com/robotedu/api/   unitaires, contrôleurs (MockMvc), intégration
    session-manager/
      pom.xml  Dockerfile
      src/main/java/com/robotedu/sessions/
        SessionManagerApplication.java
        api/                  API interne
        docker/               création et suppression des conteneurs (docker-java)
        limits/               profils de ressources
        reconciler/           réconcilie l'état réel et l'état en base
        workspace/            copie des fichiers vers et depuis le conteneur
      src/test/java/
    realtime-proxy/
      pom.xml  Dockerfile
      src/main/java/com/robotedu/proxy/
        ProxyApplication.java
        auth/  routing/
        rosfilter/            liste blanche des opérations rosbridge
        terminal/             relais du terminal
        limits/               débit et taille de message
      src/test/java/
    grading/                  (au début : simple package de api)
      pom.xml  Dockerfile
      src/main/java/com/robotedu/grading/
        worker/               consomme attempt.submitted
        graders/              PoseGoalGrader, TopicShapeGrader, MapCompareGrader, QuizGrader
      src/main/resources/rubrics/   *.yaml barèmes
      src/test/java/

  runtime/                    environnement d'exercice : le seul endroit où l'on écrit du Python
    Dockerfile              ROS 2 Jazzy + PyBullet + rosbridge + terminal
    entrypoint.sh           lance sim_node, rosbridge, terminal
    terminal/               configuration du serveur de terminal
    ros2_ws/src/
      robot_edu_description/   urdf/  config/
      robot_edu_worlds/        worlds/*.json  world.schema.json
      robot_edu_sim/           robot_edu_sim/{sim_node,lidar,physics,noise}.py  test/
      robot_edu_bringup/       launch/  params/
      robot_edu_eval/          robot_edu_eval/{pose_probe,topic_recorder}.py
      robot_edu_examples/      solutions des leçons (réservées à la vérification)

  web/                      voir le chapitre 14

  content/
    courses/
      intro-ros2/
        course.yaml         titre, niveau, ordre des leçons
        lessons/
          01-premier-node/  lesson.fr.md  lesson.en.md  lesson.yaml  starter/
          02-topics-cmd-vel/…
          03-lidar-laserscan/…
    assets/                 images des leçons

  data/
    manifests/              un fichier YAML par dataset (source, licence, somme de contrôle)
    README.md               les fichiers lourds ne sont pas dans Git : script de téléchargement

  infra/
    caddy/Caddyfile
    prometheus/prometheus.yml
    grafana/dashboards/
    scripts/                dev-up.sh  seed.sh  backup.sh  wait-for.sh
    server/                 notes d'installation du serveur Linux

  tools/
    generate_api_types.sh   OpenAPI vers types TypeScript
    (les mondes et les leçons sont validés par des tests JUnit dans services/api)

  tests/
    integration/            scénarios avec Docker Compose
    e2e/                    Playwright : parcours complet
    load/                   charge : plusieurs sessions simultanées
```

### 15.2 Rôle et propriétaire de chaque dossier

| Dossier | Contenu | Propriétaire | Relecteur |
| --- | --- | --- | --- |
| `runtime/` | Image et workspace ROS du simulateur | A | D |
| `web/` | Application React | B | C |
| `services/api/` | API modulaire | C | B |
| `services/session-manager/` | Gestionnaire de conteneurs | C | A |
| `services/realtime-proxy/` | Proxy WebSocket | C | B |
| `services/grading/` | Vérification des tentatives | A | D |
| `content/` | Cours, leçons, code de départ | D | A |
| `data/` | Manifestes de datasets | D | A |
| `infra/`, `compose*.yaml`, `.gitlab-ci.yml` | Déploiement et CI | D | C |
| `docs/contracts.md`, `docs/frames.md` | Contrats | Tous | Tous |
| `tests/` | Intégration, bout en bout, charge | D | l'auteur du composant testé |

Le fichier `.gitlab/CODEOWNERS` encode ce tableau : GitLab demande automatiquement la relecture du relecteur concerné.

### 15.3 Branches, commits et Merge Requests

- **Branche principale `main`** : protégée, toujours déployable, aucune écriture directe.
- **Branches de travail** : `feature/<zone>-<sujet>`, `fix/<zone>-<sujet>`, `docs/<sujet>` ; par exemple `feature/viewer3d-scan-layer`. Durée de vie courte (3 jours au plus).
- **Commits** : format Conventional Commits — `feat(viewer3d): add scan layer`, `fix(api): reject expired session`, `docs: update contracts`. Les types utilisés : `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `ci`.
- **Merge Request** : une tâche, un ticket lié (`Closes #12`), un relecteur obligatoire, pipeline vert, décrire comment tester. Fusion par *squash* pour garder un historique lisible.
- **Versions** : étiquettes `v0.1.0`, `v0.2.0`… posées à la fin de chaque phase de la feuille de route (section 7) ; image Docker étiquetée avec la même version.
- **Contrats** : une MR qui modifie `docs/contracts.md` ou un schéma exige l'approbation de chaque propriétaire concerné.

### 15.4 Outils de confort

| Outil | Rôle |
| --- | --- |
| `Makefile` | `make up` lance Compose ; `make test` lance tous les tests ; `make lint` ; `make seed` ; `make types` régénère les types du front depuis OpenAPI |
| `pre-commit` | Formate et vérifie avant chaque commit (Spotless pour Java, Ruff pour le Python du runtime, ESLint et Prettier pour TypeScript, détection de secrets) |
| `.editorconfig` | Indentation et fins de ligne identiques pour tous |
| Modèles de tickets et de MR | Rappel du ticket, des tests et des impacts sur les contrats |
| `devcontainer` (optionnel) | Environnement de développement identique pour les 4 étudiants |

### 15.5 Alternative : plusieurs dépôts

| Critère | Monorepo (recommandé) | Plusieurs dépôts |
| --- | --- | --- |
| Changement touchant API et front | Une MR | Deux MR à coordonner |
| Contrats partagés | Un seul fichier | Dupliqués ou paquet séparé |
| CI | Une pipeline avec règles par dossier | Une pipeline par dépôt |
| Droits d'accès fins | Moins précis (CODEOWNERS) | Précis |
| Adapté à | Équipe de 4, projet de 4 mois | Équipes séparées, cycles de vie différents |

Si l'équipe décide malgré tout de séparer, découper en trois : `robot-edu-platform` (web + services + infra), `robot-edu-runtime` (simulateur) et `robot-edu-content` (cours). Les contrats vivent alors dans `robot-edu-platform` et sont consommés par les deux autres comme sous-module Git.

### 15.6 Premier jour dans le dépôt

1. Cloner le dépôt dans le système de fichiers Linux (WSL 2 sous Windows), pas sous `/mnt/c`.
2. Copier `.env.example` en `.env` et vérifier les valeurs.
3. `make up` : Compose construit et démarre `db`, `redis`, `api`, `web`.
4. `make seed` : charge le cours d'exemple, les mondes et le compte de test.
5. Ouvrir `http://localhost:5173` et se connecter avec le compte de test.
6. `make test` : tous les tests doivent passer avant de commencer à travailler.

## 16. Infrastructure : Docker, Compose, reverse proxy et CI/CD

Référence de D (avec C pour le gestionnaire de sessions). L'objectif est qu'un nouveau membre lance toute la plateforme avec `make up`, et que le serveur de démonstration soit une copie du même Compose avec des limites plus strictes.

### 16.1 Images Docker

| Image | Base | Contenu | Utilisateur |
| --- | --- | --- | --- |
| `robot-edu/api` | `eclipse-temurin:21-jre` | Spring Boot, Flyway, dépendances Maven (pom.xml) | non-root |
| `robot-edu/session-manager` | `eclipse-temurin:21-jre` | docker-java, API interne | non-root, accès au socket via un proxy restreint |
| `robot-edu/realtime-proxy` | `eclipse-temurin:21-jre` | Spring WebSocket, `Java-WebSocket` | non-root |
| `robot-edu/grading` | `eclipse-temurin:21-jre` | worker, bibliothèques de vérification | non-root |
| `robot-edu/web` | `node:22` (build) puis `caddy` ou `nginx` (service) | fichiers statiques du front | non-root |
| `robot-edu/runtime` | `ros:jazzy-ros-base` | ROS 2, PyBullet, rosbridge, serveur de terminal, `sim_node` | `student` (UID 1000) |

**Dockerfile du runtime (squelette)**

```dockerfile
FROM ros:jazzy-ros-base
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN apt-get update && apt-get install -y --no-install-recommends \
      python3-pip python3-colcon-common-extensions build-essential \
      ros-jazzy-rosbridge-server ros-jazzy-tf2-ros ros-jazzy-robot-state-publisher \
      ros-jazzy-slam-toolbox ros-jazzy-nav2-map-server ttyd \
    && rm -rf /var/lib/apt/lists/*
RUN pip3 install --break-system-packages pybullet numpy
RUN useradd -m -u 1000 student
COPY ros2_ws /opt/robot_edu_ws
RUN source /opt/ros/jazzy/setup.bash && cd /opt/robot_edu_ws && colcon build --symlink-install
COPY entrypoint.sh /entrypoint.sh
USER student
WORKDIR /home/student
EXPOSE 9090 7681 8081
ENTRYPOINT ["/entrypoint.sh"]
```

Le script `entrypoint.sh` charge ROS, lance `sim_node` avec les paramètres de la session, puis rosbridge sur le port 9090, puis le terminal (`ttyd`) sur le port 7681, et expose un petit contrôle de santé sur le port 8081.

### 16.2 Docker Compose (développement)

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: robot_edu
    volumes: [ "dbdata:/var/lib/postgresql/data" ]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
  redis:
    image: redis:7
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s }
  api:
    build: ./services/api
    env_file: .env
    depends_on: { db: { condition: service_healthy }, redis: { condition: service_started } }
    ports: [ "127.0.0.1:8000:8000" ]
  session-manager:
    build: ./services/session-manager
    env_file: .env
    depends_on: [ redis, db ]
    volumes: [ "/var/run/docker.sock:/var/run/docker.sock" ]   # voir la note ci-dessous
    networks: [ internal, sessions ]
  realtime-proxy:
    build: ./services/realtime-proxy
    env_file: .env
    depends_on: [ redis ]
    ports: [ "127.0.0.1:8100:8100" ]
    networks: [ internal, sessions ]
  grading:
    build: ./services/grading
    env_file: .env
    depends_on: [ redis, db ]
  web:
    build: ./web
    ports: [ "127.0.0.1:5173:5173" ]
  runtime-image:
    build: ./runtime
    image: robot-edu/runtime:dev
    profiles: [ "build-only" ]            # construite par Compose, lancée par session-manager
networks:
  internal: {}
  sessions: { internal: true }            # réseau des conteneurs de session, sans accès Internet
volumes:
  dbdata: {}
```

**Note sur le socket Docker.** En développement, monter `/var/run/docker.sock` dans `session-manager` est le plus simple, mais cela donne à ce conteneur le contrôle total de l'hôte. Pour la démo, deux options : (1) lancer `session-manager` directement sur l'hôte sous un utilisateur dédié ; (2) mettre un proxy de socket filtrant (par exemple `tecnativa/docker-socket-proxy`) qui n'autorise que la création, la lecture et la suppression de conteneurs. Aucun autre conteneur ne monte le socket.

### 16.3 Limites appliquées à chaque conteneur de session

```java
// services/session-manager/.../docker/ContainerFactory.java (extrait, docker-java)
HostConfig host = HostConfig.newHostConfig()
    .withNetworkMode("session-net-" + sessionId)      // un réseau interne par session
    .withReadonlyRootfs(true)
    .withMemory(768L * 1024 * 1024).withMemorySwap(768L * 1024 * 1024)
    .withNanoCPUs(1_000_000_000L)                      // 1 CPU
    .withPidsLimit(256L)
    .withCapDrop(Capability.ALL)
    .withSecurityOpts(List.of("no-new-privileges:true"))
    .withTmpFs(Map.of("/tmp", "size=64m", "/home/student", "size=256m,uid=1000"));

CreateContainerResponse created = docker.createContainerCmd(settings.runtimeImage()) // liste blanche, jamais venue de l'utilisateur
    .withName("session-" + sessionId)
    .withUser("1000:1000")
    .withEnv("ROS_DOMAIN_ID=" + domainId, "SESSION_ID=" + sessionId, "WORLD=" + worldKey)
    .withLabels(Map.of("robot-edu.session", sessionId.toString(), "robot-edu.expires", expiresIso))
    .withHostConfig(host)                              // pas d'auto-remove : supprimé après collecte des fichiers
    .exec();
docker.startContainerCmd(created.getId()).exec();
```

| Ressource | Valeur de départ | À ajuster après mesure |
| --- | --- | --- |
| Mémoire | 768 Mo | Oui, dès la semaine 3 |
| CPU | 1 cœur | Oui |
| Processus | 256 | Non |
| Durée de session | 45 minutes, prolongeable 2 fois | Oui |
| Disque temporaire | 256 Mo | Non |
| Sessions simultanées | Selon le serveur (voir 16.7) | Oui |

### 16.4 Reverse proxy (Caddy)

```text
{$SITE_ADDRESS} {
    encode zstd gzip
    header {
        Strict-Transport-Security "max-age=31536000"
        X-Content-Type-Options nosniff
        Referrer-Policy strict-origin-when-cross-origin
        Content-Security-Policy "default-src 'self'; connect-src 'self' wss://{$SITE_ADDRESS}; img-src 'self' data:; style-src 'self' 'unsafe-inline'; worker-src 'self' blob:"
    }
    handle /api/* { reverse_proxy api:8000 }
    handle /ws/*  { reverse_proxy realtime-proxy:8100 }
    handle        { root * /srv/web ; try_files {path} /index.html ; file_server }
}
```

La politique `Content-Security-Policy` doit autoriser les *workers* de Monaco ; elle se teste dès la semaine 7 avec l'éditeur.

### 16.5 Variables d'environnement

| Variable | Utilisée par | Exemple | Secrète |
| --- | --- | --- | --- |
| `DB_PASSWORD` | `db`, `api`, `session-manager`, `grading` | — | Oui |
| `DATABASE_URL` | services Java | `postgresql://postgres:…@db/robot_edu` | Oui |
| `REDIS_URL` | services Java | `redis://redis:6379/0` | Non |
| `JWT_SECRET` | `api` | chaîne aléatoire de 64 octets | Oui |
| `SESSION_TOKEN_SECRET` | `api`, `realtime-proxy` | chaîne aléatoire | Oui |
| `INTERNAL_API_KEY` | `api`, `session-manager` | chaîne aléatoire | Oui |
| `RUNTIME_IMAGE` | `session-manager` | `robot-edu/runtime:0.3.0` | Non |
| `MAX_CONCURRENT_SESSIONS` | `session-manager` | `8` | Non |
| `SESSION_TTL_MINUTES` | `session-manager` | `45` | Non |
| `SITE_ADDRESS` | Caddy | `robot-edu.example.fr` | Non |

Les valeurs secrètes ne sont jamais dans Git : `.env` est ignoré, `.env.example` liste les noms avec des valeurs factices, et GitLab conserve les valeurs de CI dans ses variables protégées.

### 16.6 Pipeline CI GitLab

```yaml
stages: [lint, test, build, deploy]

lint:java:
  stage: lint
  image: maven:3.9-eclipse-temurin-21
  script: [ mvn -q -f services/pom.xml spotless:check ]
  rules: [ { changes: [ "services/**/*" ] } ]

lint:python:            # nœuds ROS 2 du runtime uniquement
  stage: lint
  image: python:3.12-slim
  script: [ pip install ruff, ruff check runtime ]
  rules: [ { changes: [ "runtime/**/*" ] } ]

lint:web:
  stage: lint
  image: node:22
  script: [ cd web, npm ci, npm run lint, npx tsc --noEmit ]
  rules: [ { changes: [ "web/**/*" ] } ]

test:api:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  services: [ "postgres:16", "redis:7" ]
  script: [ mvn -f services/pom.xml verify ]

test:runtime:
  stage: test
  image: ros:jazzy-ros-base
  script:
    - cd runtime/ros2_ws && colcon build && colcon test && colcon test-result --verbose

test:web:
  stage: test
  image: node:22
  script: [ cd web, npm ci, npm test -- --run ]

build:images:
  stage: build
  image: docker:27
  services: [ docker:27-dind ]
  script:
    - docker compose build
  rules: [ { if: '$CI_COMMIT_BRANCH == "main"' } ]

deploy:demo:
  stage: deploy
  script: [ ./infra/scripts/deploy.sh ]
  rules: [ { if: '$CI_COMMIT_TAG' } ]
  when: manual
```

Règles : les jobs de lint et de test s'exécutent sur chaque Merge Request ; la construction des images sur `main` ; le déploiement de démo est manuel et seulement sur une étiquette de version. Les tests d'intégration (`tests/integration/`) tournent la nuit ou à la demande, car ils démarrent de vrais conteneurs.

### 16.7 Dimensionnement du serveur de démonstration

Le coût dominant est le nombre de sessions de simulation. Estimation à vérifier par mesure (section 13.8) :

| Hypothèse par session | Valeur |
| --- | --- |
| CPU moyen | 0,25 à 0,5 cœur |
| Mémoire | 400 à 700 Mo |
| Débit WebSocket | 50 à 200 Ko/s |

Pour un serveur de 8 cœurs et 16 Go de mémoire, en réservant 4 Go et 2 cœurs aux services permanents, on peut viser **10 à 12 sessions simultanées** en conservant de la marge. La valeur de `MAX_CONCURRENT_SESSIONS` se fixe après le test de charge de la semaine 13, pas avant.

### 16.8 Observabilité

- **Journaux** : JSON sur la sortie standard avec `timestamp`, `level`, `service`, `request_id`, `session_id` ; collectés par `docker compose logs` au début, par Loki ensuite si le temps le permet.
- **Métriques** (Prometheus) : `sessions_active`, `session_start_seconds`, `session_failures_total`, `ws_connections`, `ws_messages_rejected_total`, `container_cpu_percent`, `container_memory_bytes`, `grading_duration_seconds`.
- **Tableaux de bord Grafana** : (1) santé de la plateforme, (2) sessions et capacité, (3) erreurs de l'API.
- **Alertes simples** : session qui ne démarre pas après 60 s, plus de 90 % de la capacité, disque presque plein.

### 16.9 Exploitation

| Opération | Commande ou procédure | Fréquence |
| --- | --- | --- |
| Démarrer | `docker compose up -d` | — |
| Mettre à jour | `git pull`, `docker compose build`, `docker compose up -d`, migrations Flyway | à chaque version |
| Sauvegarder la base | `pg_dump` vers un fichier daté, copié hors du serveur | quotidienne |
| Nettoyer les conteneurs orphelins | tâche de réconciliation du gestionnaire + `docker ps --filter label=robot-edu.session` | automatique, toutes les 30 s |
| Arrêter sans perdre les données | `docker compose down` (sans `-v`) | — |
| Surveiller le disque | `docker system df`, suppression des images inutilisées | hebdomadaire |

## 17. Sécurité : modèle de menace et mesures

Le risque principal vient d'une fonction voulue : l'étudiant exécute du code dans un shell sur notre serveur. Ce chapitre liste ce qu'on protège, qui peut attaquer, et les mesures par couche. Référent : C, avec D pour la vérification.

### 17.1 Ce qu'on protège

| Actif | Pourquoi c'est important |
| --- | --- |
| Serveur hôte et son réseau | Une évasion de conteneur compromet tout le reste |
| Comptes et données personnelles | Obligation légale (RGPD) et confiance |
| Code et progression des étudiants | Ne doit pas être lisible par un autre étudiant |
| Corrigés et barèmes | Sinon la vérification n'a plus de sens |
| Disponibilité de la démo | Un étudiant ne doit pas pouvoir saturer le serveur |

### 17.2 Acteurs et scénarios d'abus

| Scénario | Exemple concret | Mesure principale |
| --- | --- | --- |
| Étudiant curieux ou malveillant dans son conteneur | Tente de lire le socket Docker, d'élever ses droits, de scanner le réseau | Conteneur non privilégié, `cap_drop=ALL`, `no-new-privileges`, réseau interne sans Internet, pas de socket monté |
| Consommation abusive de ressources | Boucle infinie, minage, bombe de processus | Limites de CPU, mémoire, processus ; durée maximale ; surveillance |
| Accès à la session d'un autre | Deviner un `sessionId`, rejouer un jeton | UUID aléatoires, jeton court lié à une session et à un utilisateur, contrôle de propriété à chaque appel |
| Injection via le navigateur | Message rosbridge hostile, JSON malformé | Liste blanche d'opérations et de topics, validation de schéma, limites de taille et de débit |
| Attaque web classique | XSS dans le texte d'une leçon, CSRF, injection SQL | Markdown nettoyé, CSP stricte, cookies `SameSite`, requêtes paramétrées (ORM) |
| Vol ou devinette de mots de passe | Force brute sur `/auth/login` | Argon2id, limitation de débit, verrouillage temporaire |
| Fuite de secrets | `.env` ou jeton dans Git | `.gitignore`, détection de secrets au commit, variables CI protégées |
| Compromission de la chaîne logicielle | Paquet npm ou pip malveillant | Versions verrouillées, audit de dépendances en CI, images épinglées |
| Triche sur les scores | Modifier la requête `POST /attempts` | Le score est calculé par `grading` à partir de l'état de la simulation, jamais fourni par le client |

### 17.3 Mesures par couche

**Navigateur et site**

- En-têtes : `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options`, `Referrer-Policy`.
- Aucun HTML non filtré : le Markdown passe par `rehype-sanitize` côté front et un nettoyage côté serveur.
- Les jetons ne sont pas stockés dans `localStorage` ; les cookies d'accès sont `HttpOnly`.

**API**

- Validation stricte de toutes les entrées avec des Bean Validation (@Valid, @NotBlank, @Size) sur les DTO ; limites de taille de corps.
- Contrôle d'accès à chaque route : un utilisateur ne lit que ses sessions, tentatives et fichiers ; vérifier la propriété dans le service, pas seulement dans la route.
- Erreurs génériques côté client (pas de trace d'exécution), détails dans les journaux avec `requestId`.
- Clé d'API interne (`INTERNAL_API_KEY`) entre `api` et `session-manager`, sur un réseau Docker non exposé.

**Proxy temps réel**

- Jeton de session signé, à durée courte (5 minutes pour se connecter, valable seulement pour la session), vérifié au premier message.
- Liste blanche des opérations rosbridge, des topics et des types ; fréquence et taille de message limitées ; vitesses de `/cmd_vel` plafonnées.
- Fermeture immédiate à l'arrêt de la session.
- Un seul navigateur par session et par type de connexion (une nouvelle connexion remplace l'ancienne).

**Conteneur de session**

- Paramètres de la section 16.3 : non-root, système de fichiers en lecture seule, `tmpfs` limités, ressources bornées, aucune capacité Linux, pas de nouveaux privilèges.
- Réseau Docker interne : le conteneur ne joint ni Internet ni les autres conteneurs de session (un réseau Docker par session, auquel le gestionnaire rattache uniquement le proxy temps réel).
- `ROS_DOMAIN_ID` distinct par session, en plus de l'isolation réseau (le domaine seul n'est pas une frontière de sécurité).
- Profil `seccomp` par défaut de Docker ; `AppArmor` activé sur l'hôte si disponible.
- Les corrigés ne sont jamais dans l'image `runtime` ; ils vivent dans `grading` et dans `robot_edu_eval`.

**Gestionnaire de sessions**

- Seul composant avec accès à Docker, derrière un proxy de socket filtrant ou sur l'hôte sous un utilisateur dédié.
- N'accepte que des paramètres de la liste blanche (image, limites) ; valide les clés de monde et de robot contre le catalogue.
- Réconciliation périodique : tout conteneur portant l'étiquette `robot-edu.session` mais sans session valide est supprimé.

**Données**

- PostgreSQL non exposé à l'extérieur ; mot de passe fort ; compte applicatif sans droit de créer ou supprimer des bases.
- Sauvegardes chiffrées ou stockées hors du serveur ; restauration testée.
- Jetons de rafraîchissement stockés sous forme de condensé (hash), jamais en clair.

**Chaîne logicielle**

- Fichiers de verrouillage commités (`package-lock.json`, `uv.lock` ou `requirements.txt` avec condensés).
- Audit en CI : `npm audit`, `pip-audit`, analyse des images (Trivy) à chaque Merge Request vers `main` si le temps le permet.
- Images `runtime` épinglées par version ; aucune image fournie par un utilisateur.

### 17.4 Données personnelles (RGPD)

| Point | Décision |
| --- | --- |
| Données collectées | E-mail, pseudo, langue, progression, code écrit dans les leçons |
| Base légale | Exécution du service d'enseignement ; informer l'utilisateur dans une page de confidentialité |
| Minimisation | Pas de suivi publicitaire ; pas de données de localisation ; journaux sans code ni mot de passe |
| Durées de conservation | Voir 11.4 ; journaux d'accès 30 jours |
| Droits | Export et suppression du compte (`DELETE /users/me`) ; contact indiqué sur le site |
| Hébergement | Serveur de l'université ou hébergeur dans l'Union européenne |

Ce projet universitaire n'est pas une analyse juridique ; la page de confidentialité doit être validée par l'encadrant avant toute ouverture à des utilisateurs externes.

### 17.5 Liste de contrôle avant la démonstration

- [ ] Aucun secret dans l'historique Git (`git log -p` filtré, outil de détection de secrets).
- [ ] Seuls les ports 80 et 443 sont ouverts depuis l'extérieur ; 9090, 7681, 5432, 6379 ne le sont pas.
- [ ] Un conteneur de session ne peut pas joindre Internet, ni le conteneur voisin, ni le socket Docker (test manuel à l'intérieur du terminal).
- [ ] Une boucle infinie ou une bombe de processus dans le terminal est contenue par les limites.
- [ ] Un jeton de session d'un utilisateur ne fonctionne pas pour la session d'un autre.
- [ ] Une session expirée est supprimée avec ses conteneurs et ses fichiers temporaires.
- [ ] Les limites de débit répondent `429` comme prévu.
- [ ] La sauvegarde de la base a été restaurée une fois avec succès.
- [ ] La page de confidentialité et la page « À propos » (limites, licences) sont publiées.

## 18. Tests et qualité

Référent : D, avec chaque propriétaire responsable des tests de son composant. Un composant n'est pas « fini » sans ses tests.

### 18.1 Pyramide de tests

| Niveau | Outil | Ce qu'on teste | Quand | Qui |
| --- | --- | --- | --- | --- |
| Unitaires Java (JUnit 5) | `mvn test` | Fonctions de l'API, cinématique, LiDAR, conversion de barèmes | À chaque commit (CI) | C, A |
| Unitaires TypeScript | Vitest | `frames.ts`, validation des messages, stores | À chaque commit | B |
| Composants | Testing Library | Pages et panneaux avec des données simulées | À chaque MR | B |
| ROS | `launch_testing`, `colcon test` | `sim_node` répond à `/cmd_vel` et publie `/odom`, `/scan` | À chaque MR touchant `runtime/` | A |
| Contrats API | Schemathesis ou tests contre OpenAPI | Réponses conformes au schéma | À chaque MR touchant l'API | C |
| Intégration | `mvn verify` + Testcontainers | Création d'une session réelle, connexion, arrêt, nettoyage | La nuit et avant chaque version | D |
| Bout en bout | Playwright | Parcours complet dans un vrai navigateur | Avant chaque version | D et B |
| Charge | Locust ou k6 | N sessions simultanées, débit WebSocket | Semaines 12 et 13 | D et C |
| Sécurité | Liste du 17.5 + tests manuels | Évasion, isolation, jetons | Avant la démo | C et D |

### 18.2 Tests essentiels par composant

**Simulateur (A)**

- Une commande `v = 0,2 m/s` pendant 5 s fait avancer le robot d'environ 1 m (tolérance 5 %).
- Une commande `ω = 0,5 rad/s` pendant π/0,5 s fait tourner le robot d'environ 180°.
- Dans le monde `simple-room`, la distance mesurée dans l'axe d'un mur connu est celle calculée à la main (± 1 cm sans bruit).
- Deux exécutions avec la même graine donnent la même trajectoire.
- Sans nouvelle commande pendant 0,5 s, le robot s'arrête.
- Le message `LaserScan` a `angle_increment` cohérent avec le nombre de rayons.

**API (C)**

- Un utilisateur ne peut pas lire, arrêter ou soumettre sur la session d'un autre (`403` ou `404`).
- Deux `POST /sessions` avec la même `Idempotency-Key` ne créent qu'une session.
- Le quota de sessions simultanées est respecté, même avec deux requêtes en parallèle.
- `DELETE /sessions/{id}` appelé deux fois renvoie le même résultat.
- Une tentative soumise sur une session arrêtée est refusée (`409`).
- Un exercice non publié n'apparaît pas dans le catalogue public.

**Gestionnaire de sessions (C)**

- Un conteneur créé porte bien les limites et l'utilisateur attendus (inspection du conteneur).
- Un conteneur dont la session a expiré est supprimé en moins de 60 s.
- Un conteneur orphelin (sans session en base) est supprimé par la réconciliation.
- Quand la capacité est pleine, la création échoue proprement avec `capacity_full`.

**Proxy temps réel (C)**

- Sans jeton ou avec un jeton expiré, la connexion est fermée en moins de 5 s.
- `subscribe` sur un topic hors liste blanche est refusé.
- `publish` sur `/cmd_vel` avec une vitesse excessive est plafonné ou refusé.
- Un message de plus de 64 Ko est rejeté.
- À l'arrêt de la session, toutes ses connexions sont fermées.

**Front (B)**

- `rosToThreePosition(1, 0, 0)` donne `(1, 0, 0)` ; `rosToThreePosition(0, 1, 0)` donne `(0, 0, -1)`.
- Un `LaserScan` avec `NaN`, `inf` et des valeurs hors portée n'ajoute aucun point invalide.
- Perte de la connexion : la page affiche le bandeau puis se reconnecte avec délai croissant.
- Rafraîchir la page pendant une session la retrouve.

**Vérification (A et D)**

- Un corrigé de référence obtient le score maximal ; un code vide obtient 0.
- La même tentative donne toujours le même score.
- Chaque critère du barème a au moins un test qui l'active et un qui le désactive.

### 18.3 Scénarios d'acceptation de bout en bout

| # | Scénario | Résultat attendu |
| --- | --- | --- |
| E1 | Un nouvel utilisateur s'inscrit, ouvre la leçon 1 et démarre la simulation | Session `running` en moins de 20 s ; robot visible |
| E2 | Il écrit un node qui publie `/cmd_vel` et le lance dans le terminal | Le robot avance dans la vue 3D |
| E3 | Il affiche le scan et active la couche LiDAR | Les points s'alignent sur les murs du monde |
| E4 | Il soumet l'exercice | Score affiché en moins de 15 s ; progression mise à jour |
| E5 | Il rafraîchit la page pendant la session | Session retrouvée, code conservé |
| E6 | Il laisse expirer la session | Fenêtre de fin ; conteneur supprimé ; code sauvegardé |
| E7 | Un second utilisateur tente d'ouvrir l'adresse de la session du premier | Accès refusé |
| E8 | La capacité est pleine, un utilisateur demande une session | Message clair, lecture de la leçon toujours possible |
| E9 | L'utilisateur lance un SLAM sur le monde simulé | Une carte apparaît dans la vue 3D |
| E10 | Un administrateur arrête le serveur avec `docker compose down` puis le relance | Aucune donnée perdue ; les sessions orphelines sont nettoyées |

Chaque scénario est écrit dans `docs/acceptance/` avec les étapes exactes, et automatisé avec Playwright quand c'est raisonnable (E1 à E7).

### 18.4 Objectifs de performance

| Mesure | Objectif | Comment mesurer |
| --- | --- | --- |
| Démarrage d'une session | moins de 20 s (médiane) | métrique `session_start_seconds` |
| Latence commande vers mouvement visible | moins de 300 ms | horodatage à l'envoi et à la réception de l'odométrie |
| Images par seconde de la vue 3D | au moins 50 | outils de performance du navigateur |
| Charge CPU d'une session | moins de 50 % d'un cœur | `docker stats` |
| Sessions simultanées | valeur cible du serveur sans dépasser 80 % de CPU | test de charge |
| Temps de réponse de l'API (hors sessions) | moins de 200 ms au 95e centile | journaux et métriques |

### 18.5 Définition de « terminé » pour un ticket

1. Le code est relu et fusionné par Merge Request.
2. Les tests du composant passent en CI, y compris un test pour le cas d'erreur le plus probable.
3. Les contrats modifiés sont à jour dans `docs/contracts.md`.
4. Un collègue qui n'a pas écrit le code a réussi à le lancer en suivant le README.
5. Aucune alerte de lint ni de sécurité nouvelle.

## 19. Programme pédagogique

Référent : D, avec A pour la partie ROS et simulation. Le programme suit la progression de The Construct : on apprend un concept, on le fait sur le robot, on le vérifie. Chaque leçon dure 30 à 60 minutes.

### 19.1 Structure d'une leçon

Chaque leçon est un dossier `content/courses/<cours>/lessons/<nn-nom>/` contenant :

| Fichier | Rôle |
| --- | --- |
| `lesson.fr.md`, `lesson.en.md` | Texte : objectif, théorie courte, étapes, indices |
| `lesson.yaml` | Métadonnées : titre, durée estimée, prérequis, monde, robot, topics autorisés, barème |
| `starter/` | Code de départ copié dans l'espace de travail |
| `checks/` | Paramètres des vérifications (cibles, tolérances) |

```yaml
# lesson.yaml (exemple)
id: 03-lidar-laserscan
title: { fr: "Lire un scan LiDAR", en: "Reading a LiDAR scan" }
level: beginner
estimatedMinutes: 45
prerequisites: [ "02-topics-cmd-vel" ]
world: simple-room
robot: diffbot-v1
simulator:
  lidar: { num_rays: 360, noise_std: 0.0 }
allowedTopics:
  subscribe: [ /scan, /odom, /tf ]
  publish: [ /cmd_vel ]
grading:
  rubric: lidar-intro-v1
  maxScore: 100
```

### 19.2 Cours 1 : Découvrir ROS 2 avec un robot (débutant)

| # | Leçon | Objectif d'apprentissage | Exercice | Vérification |
| --- | --- | --- | --- | --- |
| 1 | Premier node | Comprendre un node, lancer un programme ROS 2 | Écrire un node qui affiche un message chaque seconde | Le node tourne et publie ses journaux |
| 2 | Topics et `/cmd_vel` | Publier une commande de vitesse | Faire avancer le robot de 1 m en ligne droite | Pose finale à 1 m ± 10 cm |
| 3 | Odométrie | Lire `/odom` et comprendre pose et vitesse | Afficher x, y et l'angle en degrés | Réponses correctes sur une trajectoire connue |
| 4 | Tourner et se déplacer en carré | Combiner translations et rotations | Parcourir un carré de 1 m de côté | Retour à moins de 20 cm du départ |
| 5 | Lire un scan LiDAR | Interpréter `LaserScan` (angles, portées) | Afficher la distance de l'obstacle devant | Valeur à 2 cm de la distance réelle |
| 6 | Éviter un obstacle | Boucle de contrôle réactive | Avancer jusqu'à 30 cm d'un mur et s'arrêter | Distance finale 30 cm ± 5 cm, sans collision |

### 19.3 Cours 2 : Capteurs, repères et navigation de base (intermédiaire)

| # | Leçon | Objectif d'apprentissage | Exercice | Vérification |
| --- | --- | --- | --- | --- |
| 7 | Frames et TF | Comprendre `map`, `odom`, `base_link`, `laser` | Transformer un point du capteur vers le repère `odom` | Coordonnées correctes pour 3 points |
| 8 | Services et paramètres | Utiliser un service et un paramètre ROS | Changer la vitesse maximale par paramètre | Comportement conforme |
| 9 | Suivre un mur | Contrôle proportionnel avec le LiDAR | Longer un mur à 40 cm sur 4 m | Écart moyen sous 8 cm |
| 10 | Aller à un point | Navigation par odométrie | Atteindre une cible `goal` du monde | Robot dans le cercle de l'objectif |
| 11 | Bruit et dérive | Comprendre pourquoi l'odométrie dérive | Comparer odométrie et vérité terrain avec bruit activé | Rapport d'écart correct |

### 19.4 Cours 3 : Cartographie et SLAM (intermédiaire)

| # | Leçon | Objectif d'apprentissage | Exercice | Vérification |
| --- | --- | --- | --- | --- |
| 12 | Grille d'occupation | Lire `OccupancyGrid` (résolution, origine) | Compter les cellules libres d'une carte donnée | Nombre exact |
| 13 | Lancer un SLAM | Utiliser `slam_toolbox` avec le robot simulé | Cartographier la pièce en pilotant le robot | Carte proche de la référence (IoU ≥ 0,7) |
| 14 | Qualité d'une carte | Effets du bruit et de la vitesse sur la carte | Obtenir la meilleure carte possible dans le labyrinthe | IoU ≥ 0,8 |
| 15 | Sauvegarder et réutiliser une carte | `map_saver` et `map_server` | Sauvegarder la carte puis la recharger | Fichier de carte valide |

Les cours 2 et 3 sont **des cibles** ; le MVP exige au moins 3 à 5 leçons complètes (par exemple 1, 2, 5, 6 et 13). La priorité va aux leçons qui montrent la simulation 3D et le LiDAR.

### 19.5 Types de vérification

| Type | Principe | Exemple | Entrée fournie par la simulation |
| --- | --- | --- | --- |
| `pose_goal` | La pose finale est proche d'une cible | Leçons 2, 4, 10 | Pose de la base (vérité terrain) |
| `topic_shape` | Un topic est publié avec le bon type et la bonne cadence | Leçon 1 | Enregistrement des messages |
| `value_check` | Une valeur affichée ou retournée est correcte | Leçons 3, 5 | Réponse saisie, valeur de référence |
| `behavior` | Le comportement respecte des contraintes (distance, collisions) | Leçons 6, 9 | Trajectoire enregistrée |
| `map_compare` | La carte produite ressemble à la référence | Leçons 13, 14 | `OccupancyGrid` exportée |
| `quiz` | Questions à choix | Leçons de théorie | Réponses |

### 19.6 Exemple de barème (`lidar-intro-v1`)

| Critère | Points | Type | Détail |
| --- | --- | --- | --- |
| Le node s'abonne à `/scan` | 20 | `topic_shape` | Un abonné à `/scan` détecté dans la session |
| Distance frontale correcte | 40 | `value_check` | Écart ≤ 2 cm avec la valeur de référence |
| Valeurs invalides ignorées | 20 | `behavior` | Aucune exception sur un scan contenant des `inf` |
| Réponse à la question de cours | 20 | `quiz` | Bonne réponse sur l'angle d'un indice de rayon |

Le retour (`feedback`) de chaque critère est rédigé pour aider à progresser (par exemple : « Votre distance est de 1,92 m, la valeur attendue est de 1,50 m : vérifiez l'indice choisi dans `ranges`. »), pas seulement pour annoncer l'échec.

### 19.7 Datasets

Les leçons de départ n'ont pas besoin de dataset enregistré : le simulateur suffit. Un jeu rejoué est ajouté pour deux usages : montrer un LiDAR **réel** dans la vue 3D (avec le badge « Dataset rejoué ») et fournir un nuage de points 3D. Pour chaque jeu, le manifeste dans `data/manifests/` indique :

| Champ | Exemple |
| --- | --- |
| `key`, `version` | `lab-corridor`, 1 |
| `source` | URL du jeu d'origine et organisme |
| `license` | Licence qui permet la redistribution ; à vérifier avant tout ajout |
| `sensor` | Modèle du capteur |
| `frames` | `map`, `odom`, `base_link`, `laser` |
| `rate_hz`, `duration_s` | 10, 120 |
| `checksum` | SHA-256 du fichier |
| `size_mb` | Taille ; au-delà de 100 Mo, ne pas mettre dans Git |

## 20. Backlog et planning semaine par semaine

Ce chapitre transforme la feuille de route du chapitre 7 en tâches pour chaque personne. Les estimations sont en tailles (S ≤ 1 jour, M = 2 à 3 jours, L = 4 à 5 jours) et servent à détecter les semaines trop chargées, pas à promettre une date.

### 20.1 Planning par semaine et par personne

| Semaine | A : simulateur | B : interface | C : API et sessions | D : contenu, infra, tests |
| --- | --- | --- | --- | --- |
| 1 | ROS 2 dans Docker ; test PyBullet sans interface ; URDF minimal | Projet React + Vite ; scène Three.js avec cube animé | Spring Boot + `/health` ; PostgreSQL ; première migration | Dépôt GitLab, CI de base ; `docs/contracts.md` et `frames.md` avec tous |
| 2 | `sim_node` minimal : `/cmd_vel` → `/odom` | Connexion rosbridge ; robot affiché avec pose de test | `POST /sessions` simulé ; modèle de données de base | Compose complet ; procédure de vérification ; **tranche verticale (fin S2)** |
| 3 | Monde JSON v1 ; murs en collision ; `/tf` | `urdf-loader` ; `WorldLayer` depuis le JSON | `auth` (inscription, connexion, cookies) | Squelette du cours 1 ; schéma JSON du monde ; première mesure de CPU |
| 4 | LiDAR 2D `/scan` ; temps simulé `/clock` | `ScanLayer` ; conversion de repères testée | `catalog` : cours, leçons, exercices ; `content` | Leçons 1 à 3 écrites ; tests unitaires du simulateur |
| 5 | Image `runtime` complète ; `entrypoint.sh` ; santé | Pages catalogue, cours, connexion | `session-manager` : création et suppression d'un conteneur | `compose` avec réseaux ; tests d'intégration de démarrage |
| 6 | Paramètres par exercice (YAML) ; bruit LiDAR optionnel | Barre de session ; états du front ; reconnexion | `realtime-proxy` : jeton, liste blanche, routage | Test d'évasion de base ; mesures de latence |
| 7 | `robot_state_publisher` ; `/joint_states` | Monaco : éditeur et enregistrement automatique | `workspace` : fichiers de l'étudiant ; copie vers le conteneur | Politique CSP testée avec Monaco ; leçons 4 et 5 |
| 8 | Terminal `ttyd` dans l'image ; arrêt propre | Terminal xterm.js ; page de leçon assemblée | Proxy du terminal ; expiration des sessions | Scénarios E1 à E3 automatisés |
| 9 | `slam_toolbox` dans l'image ; carte publiée | `MapLayer` ; `TrailLayer` | `progress` : tentatives, scores | Mondes 2 et 3 ; leçon 6 |
| 10 | `robot_edu_eval` : sonde de pose, enregistreur de topics | Soumission et affichage du score | `grading` : worker, `pose_goal` et `topic_shape` | Barèmes ; scénario E4 automatisé |
| 11 | Corrigés de référence ; `map_compare` | Tableau de bord et progression | Quotas, file d'attente, `capacity_full` | Leçon 13 (SLAM) ; scénarios E5 à E8 |
| 12 | Optimisation CPU du simulateur ; LiDAR 3D optionnel | Badge de source ; options de la vue 3D ; accessibilité | Prolongation de session ; export des données du compte | **Test de charge** ; **gel des fonctionnalités (fin S12)** |
| 13 | Corrections ; scripts de reproductibilité | Corrections ; performances de rendu | Durcissement sécurité ; rétention et nettoyage | Liste 17.5 ; audit de dépendances |
| 14 | Documentation du simulateur | Textes et traductions ; pages d'erreur | Réconciliation ; alertes ; sauvegarde | Recette complète E1 à E10 |
| 15 | Corrections de la recette | Corrections de la recette | Corrections de la recette | Manuel d'installation ; vidéo de démo |
| 16 | Répétition de la démo | Répétition de la démo | Sauvegarde finale, version `v1.0` | Rapport, présentation, livraison |

### 20.2 Backlog initial (tickets à créer dans GitLab)

| ID | Ticket | Responsable | Dépend de | Taille | Phase |
| --- | --- | --- | --- | --- | --- |
| T01 | Créer le dépôt, protéger `main`, CODEOWNERS | D | — | S | 1 |
| T02 | Écrire `contracts.md` et `frames.md` | Tous | — | M | 1 |
| T03 | Image Docker ROS 2 + PyBullet ; test sans interface | A | T01 | M | 1 |
| T04 | `sim_node` minimal (`/cmd_vel` → `/odom`) | A | T03 | M | 1 |
| T05 | Projet React/Vite, scène Three.js, cube | B | T01 | S | 1 |
| T06 | Client rosbridge et `topicStore` | B | T02 | M | 1 |
| T07 | Spring Boot, `/health`, PostgreSQL, Flyway | C | T01 | M | 1 |
| T08 | Endpoint de session simulé | C | T07 | S | 2 |
| T09 | Compose complet (db, redis, api, web) | D | T07 | M | 2 |
| T10 | Tranche verticale : robot qui bouge dans le navigateur | Tous | T04, T06, T08 | M | 2 |
| T11 | Format du monde JSON + schéma | A et D | T02 | M | 3 |
| T12 | Monde en PyBullet (murs, boîtes) | A | T11 | M | 3 |
| T13 | `WorldLayer` Three.js | B | T11 | M | 3 |
| T14 | Chargement de l'URDF dans Three.js | B | T04 | M | 3 |
| T15 | Module `auth` | C | T07 | L | 3 |
| T16 | LiDAR 2D et `/scan` | A | T12 | L | 4 |
| T17 | `ScanLayer` et conversion des repères | B | T16 | M | 4 |
| T18 | `catalog` et `content` | C | T15 | L | 4 |
| T19 | Leçons 1 à 3 | D | T18 | L | 4 |
| T20 | Image `runtime` et `entrypoint.sh` | A | T16 | M | 5 |
| T21 | `session-manager` : créer et supprimer un conteneur | C | T20 | L | 5 |
| T22 | Pages catalogue et connexion | B | T15, T18 | M | 5 |
| T23 | `realtime-proxy` (jeton, liste blanche) | C | T21 | L | 6 |
| T24 | Barre de session, états front, reconnexion | B | T23 | M | 6 |
| T25 | Monaco et enregistrement automatique | B | T27 | M | 7 |
| T26 | Terminal web (image, proxy, xterm.js) | A, C et B | T23 | L | 7 à 8 |
| T27 | `workspace` (fichiers de l'étudiant) | C | T21 | M | 7 |
| T28 | Page de leçon assemblée | B | T24, T25, T26 | M | 8 |
| T29 | Expiration et réconciliation des sessions | C | T21 | M | 8 |
| T30 | Scénarios E1 à E3 automatisés | D | T28 | M | 8 |
| T31 | SLAM dans l'image ; carte publiée | A | T20 | L | 9 |
| T32 | `MapLayer` et `TrailLayer` | B | T31 | M | 9 |
| T33 | `progress` et tentatives | C | T18 | M | 9 |
| T34 | `robot_edu_eval` et `grading` | A et C | T33 | L | 10 |
| T35 | Soumission et score dans l'interface | B | T33 | M | 10 |
| T36 | Barèmes et corrigés de référence | D | T34 | M | 10 à 11 |
| T37 | Quotas, file d'attente, capacité | C | T29 | M | 11 |
| T38 | Tableau de bord et progression | B | T33 | M | 11 |
| T39 | Optimisation CPU et test de charge | A et D | T37 | L | 12 |
| T40 | Durcissement sécurité et liste 17.5 | C et D | T39 | L | 13 |
| T41 | Observabilité (métriques, alertes) | D | T09 | M | 13 à 14 |
| T42 | Manuel d'installation, rapport, démo | Tous | — | L | 15 à 16 |

### 20.3 Rythme de travail et réunions

| Rituel | Fréquence | Durée | But |
| --- | --- | --- | --- |
| Point de contrats | 2 fois par semaine | 15 min | Aligner les interfaces, débloquer |
| Démo commune | Toutes les 2 semaines | 30 min | Montrer un chemin de bout en bout, pas des composants |
| Rétrospective | Toutes les 4 semaines | 30 min | Ce qui marche, ce qui bloque, ajustements |
| Revue de Merge Requests | Chaque jour | — | Une MR ne reste pas ouverte plus de 2 jours |
| Point avec l'encadrant | Selon son rythme | 30 min | Valider les choix et la démo |

### 20.4 Risques de planning et réponses

| Signal d'alerte | Réponse prévue |
| --- | --- |
| La tranche verticale n'est pas prête à la fin de la semaine 2 | Geler les nouvelles fonctions, tout le monde aide sur l'intégration |
| Le CPU par session est plus élevé que prévu (semaine 3) | Réduire rayons et cadences, baisser la physique à 120 Hz, revoir le nombre de sessions |
| Le terminal web ou le gestionnaire de sessions prend du retard | Repli : un seul conteneur partagé avec un processus par session, sans terminal (l'étudiant lance son code depuis l'éditeur) |
| Le SLAM ne converge pas | Repli : exercice de grille d'occupation écrit par l'équipe, ou rejeu d'une carte précalculée |
| Une personne indisponible | Les propriétaires de dossier ont chacun un relecteur qui connaît le code ; mettre à jour le tableau 15.2 |
| Retard global à la semaine 10 | Couper les fonctions de la section 20.5 dans l'ordre indiqué |

### 20.5 Ordre de suppression si le temps manque

1. LiDAR 3D (`PointCloud2`) et dataset rejoué.
2. Cours 2 et 3 au-delà des 3 à 5 leçons du MVP.
3. Anglais (garder le français seul).
4. Tableau de bord détaillé (garder la liste de progression).
5. Observabilité avancée (garder les journaux et `docker stats`).
6. Prolongation de session et file d'attente (garder le refus propre).

Ce qui ne se coupe jamais : session isolée, simulation 3D visible, éditeur ou terminal pour écrire du code, au moins 3 leçons vérifiées, nettoyage des conteneurs.

## 21. Annexes

### 21.1 Glossaire

| Terme | Définition |
| --- | --- |
| ROS 2 | Cadre logiciel pour programmer des robots : des programmes (nodes) échangent des messages. |
| Node | Programme ROS 2 qui fait une tâche (lire un capteur, calculer une commande). |
| Topic | Canal de messages continu entre nodes (par exemple `/scan`). |
| Service / action | Appel ponctuel avec réponse / tâche longue avec suivi et annulation. |
| DDS | Couche réseau sous ROS 2 qui fait découvrir et relier les nodes. |
| `ROS_DOMAIN_ID` | Numéro qui sépare des groupes de nodes ; ce n'est pas une protection de sécurité. |
| rosbridge | Pont qui expose les topics ROS en JSON par WebSocket pour le navigateur. |
| URDF | Fichier XML qui décrit un robot : liens, articulations, formes. |
| TF (transformations) | Arbre de repères (`map`, `odom`, `base_link`, `laser`) reliant les positions entre elles. |
| Odométrie | Estimation de la position du robot à partir de ses roues ; elle dérive avec le temps. |
| LiDAR | Capteur qui mesure des distances avec un faisceau laser tournant. |
| `LaserScan` | Message ROS contenant une liste de distances à des angles réguliers (plan 2D). |
| Nuage de points | Ensemble de points XYZ mesurés en 3D (`PointCloud2`). |
| SLAM | Cartographier un lieu et se localiser en même temps. |
| `OccupancyGrid` | Carte en grille : chaque cellule est libre, occupée ou inconnue. |
| Raycasting | Lancer un rayon virtuel et mesurer où il touche un objet ; sert à simuler un LiDAR. |
| PyBullet | Bibliothèque de simulation physique, utilisée ici sans interface graphique. |
| Three.js | Bibliothèque d'affichage 3D dans le navigateur ; ne simule pas la physique. |
| Monorepo | Un seul dépôt Git pour tout le projet. |
| MVP | Première version utile et livrable, sans les options. |
| Idempotent | Repéter une demande donne le même résultat qu'une seule fois. |
| Jeton court | Autorisation valable peu de temps et pour un seul usage (ici, une session). |

### 21.2 Conventions de code

- **Java** : Google Java Style, formatage Spotless, noms explicites, Javadoc courte sur les classes publiques, pas d'état statique modifiable. Les nœuds ROS 2 du runtime suivent PEP 8, formatage Ruff.
- **TypeScript** : mode strict, pas de `any` sans commentaire, composants fonctionnels, un composant par fichier.
- **ROS** : noms de topics en minuscules avec `/` ; unités SI (m, s, rad) ; `frame_id` toujours renseigné ; horodatage avec l'horloge du node.
- **API** : noms de champs en `camelCase` dans le JSON, en `snake_case` dans la base ; dates ISO 8601.
- **Journaux** : jamais de mot de passe, jeton ni code d'étudiant ; niveau `info` par défaut.
- **Documentation** : une page `README.md` par service (rôle, lancement, variables, tests).

### 21.3 Modèle de Merge Request

```markdown
## Ce qui change
(2 à 3 phrases)

## Ticket
Closes #

## Comment tester
1.
2.

## Contrats touchés
- [ ] `docs/contracts.md`  - [ ] schéma OpenAPI  - [ ] monde JSON  - [ ] aucun

## Liste de contrôle
- [ ] Tests ajoutés ou mis à jour
- [ ] Pas de secret ni de donnée personnelle
- [ ] README mis à jour si nécessaire
```

### 21.4 Checklist d'arrivée pour un membre de l'équipe

- [ ] Accès GitLab, dépôt cloné sous Linux ou WSL 2.
- [ ] Docker et Docker Compose fonctionnent (`docker run hello-world`).
- [ ] `make up` démarre la plateforme ; `make test` est vert.
- [ ] Lecture de ce document, chapitres 1 à 8, puis du chapitre de son rôle.
- [ ] Une première petite Merge Request fusionnée (correction de documentation par exemple).

### 21.5 Liens de référence

Ces liens sont donnés comme points de départ ; ils n'ont pas été rouverts lors de la rédaction de ce document, il faut vérifier qu'ils sont toujours à jour.

| Sujet | Lien |
| --- | --- |
| ROS 2 Jazzy | [docs.ros.org/en/jazzy](https://docs.ros.org/en/jazzy/) |
| rosbridge suite | [github.com/RobotWebTools/rosbridge\_suite](https://github.com/RobotWebTools/rosbridge_suite) |
| roslibjs | [github.com/RobotWebTools/roslibjs](https://github.com/RobotWebTools/roslibjs) |
| PyBullet | [pybullet.org](https://pybullet.org/) |
| Three.js | [threejs.org/docs](https://threejs.org/docs/) |
| urdf-loader | [github.com/gkjohnson/urdf-loaders](https://github.com/gkjohnson/urdf-loaders) |
| Monaco Editor | [microsoft.github.io/monaco-editor](https://microsoft.github.io/monaco-editor/) |
| xterm.js | [xtermjs.org](https://xtermjs.org/) |
| Spring Boot | [spring.io/projects/spring-boot](https://spring.io/projects/spring-boot) |
| Docker : limites de ressources | [docs.docker.com/engine/containers/resource\_constraints](https://docs.docker.com/engine/containers/resource_constraints/) |
| Caddy | [caddyserver.com/docs](https://caddyserver.com/docs/) |
| GitLab CI/CD | [docs.gitlab.com/ci](https://docs.gitlab.com/ci/) |
| The Construct (inspiration) | [theconstruct.ai](https://www.theconstruct.ai/) |

# Partie 3 — Dossiers individuels des quatre étudiants

Chaque dossier est lisible seul : une personne peut le prendre, le suivre dans l'ordre et savoir quoi livrer, à qui demander quoi et comment prouver que c'est fini. Les lettres A, B, C et D peuvent être remplacées par les prénoms.

## 22. Vue d'ensemble de la répartition

Le travail est coupé en quatre blocs qui se rejoignent par des contrats d'interface (chapitre 6). Chaque personne possède un bloc de bout en bout — code, tests, documentation — et a un suppléant qui le connaît assez pour le dépanner.

### 22.1 Les quatre blocs en une phrase

| Étudiant | Bloc | Ce qu'il livre | Dossiers du dépôt |
| --- | --- | --- | --- |
| **A** | Simulateur et ROS 2 | Un robot qui répond à `/cmd_vel` et publie `/odom`, `/scan`, `/tf` dans un monde 3D, packagé dans l'image `runtime`, plus les outils de vérification | `runtime/`, `services/grading/` (avec C) |
| **B** | Interface web et vue 3D | Le site complet : catálogue, page de leçon avec vue Three.js, éditeur, terminal, progression | `web/` |
| **C** | API, sessions et sécurité | L'API, la base, le gestionnaire de conteneurs, le proxy temps réel, les comptes, les quotas | `services/api/`, `services/session-manager/`, `services/realtime-proxy/` |
| **D** | Contenu, infrastructure et qualité | Les leçons et barèmes, le Compose, la CI, les tests d'intégration et de charge, la documentation | `content/`, `data/`, `infra/`, `tests/`, `docs/`, `compose*.yaml`, `.gitlab-ci.yml` |

### 22.2 Matrice des responsabilités

R = responsable (écrit et livre) · C = contribue (fournit une partie ou un retour) · V = relecteur (valide la Merge Request). Une case vide signifie : pas concerné.

| Livrable | A | B | C | D |
| --- | --- | --- | --- | --- |
| `contracts.md` et `frames.md` | R | R | R | R |
| Format du monde JSON et schéma | R | C |  | C |
| URDF du robot | R | C |  |  |
| `sim_node`, LiDAR, odométrie, TF | R |  |  | V |
| Image `runtime` | R |  | C | V |
| Terminal dans le conteneur | R | C | C |  |
| Vue 3D (monde, robot, scan, carte) | C | R |  |  |
| Éditeur et terminal dans le navigateur |  | R | C |  |
| Pages du site et traductions |  | R |  | C |
| API : auth, catalogue, progression, espace de travail |  | C | R |  |
| Gestionnaire de sessions | V |  | R |  |
| Proxy temps réel et liste blanche | C | V | R |  |
| Sécurité des conteneurs et des jetons | C |  | R | V |
| Vérification (`grading`, `robot_edu_eval`) | R |  | R | V |
| Leçons, barèmes, corrigés | C | C |  | R |
| Datasets et manifestes | C |  |  | R |
| Compose, CI, déploiement |  |  | C | R |
| Tests d'intégration, bout en bout, charge | C | C | C | R |
| Observabilité |  |  | C | R |
| Rapport, manuel d'installation, démo | R | R | R | R |

### 22.3 Qui livre quoi à qui, et quand

C'est le tableau à relire chaque lundi : chaque ligne est un engagement qui peut bloquer quelqu'un d'autre.

| De | Vers | Livraison | Au plus tard |
| --- | --- | --- | --- |
| A | B, C, D | `/cmd_vel` → `/odom` fonctionnel et exemple de messages | Fin de la semaine 2 |
| A | B | Format du monde JSON figé, un monde d'exemple, l'URDF | Semaine 3 |
| A | B | `/scan`, `/tf`, `/joint_states` avec exemples enregistrés | Semaine 4 |
| A | C | Image `runtime` avec ports, variables d'environnement, contrôle de santé | Semaine 5 |
| A | D | Paramètres des exercices et corrigés de référence | Semaines 9 à 11 |
| B | C | Besoins d'API de chaque page (liste des champs) | Semaine 2 |
| B | A | Retour sur les frames et les conversions (ce qui est faux à l'écran) | En continu |
| B | D | Format de rendu du Markdown des leçons (balises permises) | Semaine 3 |
| C | B | Schéma OpenAPI et types, endpoint de session simulé | Fin de la semaine 2 |
| C | B | Bloc de connexion réel (`connection`), jetons, proxy | Semaine 6 |
| C | A | Contraintes du gestionnaire : variables, ports, limites, signaux d'arrêt | Semaine 4 |
| C | D | Comptes de test, jeu de données de départ (`seed`) | Semaine 3 |
| D | Tous | Compose, CI, `make up` qui marche | Fin de la semaine 2 |
| D | C | Format de `lesson.yaml` et de barème | Semaine 3 |
| D | B | Premier cours de démonstration complet | Semaine 4 |

### 22.4 Binômes de revue et suppléants

| Personne | Relue par | Suppléant sur | Pourquoi |
| --- | --- | --- | --- |
| A | D | `runtime/`, image Docker | D connaît l'image car il la construit dans le Compose |
| B | C | `web/` (pages et client rosbridge) | C connaît les contrats qu'appelle le front |
| C | B, et A pour le gestionnaire | `services/api/` | B connaît l'API côté appel ; A connaît ce que le gestionnaire lance |
| D | C | `infra/` et CI | C connaît le déploiement des services |

Règle : une Merge Request est relue en moins de 24 heures ouvrées ; un relecteur indisponible est remplacé par son suppléant.

### 22.5 Équilibre de la charge

Nombre de tickets du backlog (section 20.2) par personne, hors les 3 tickets communs (T02, T10, T42) :

| Personne | Tickets seuls | Tickets partagés | Total d'implications | Remarque |
| --- | --- | --- | --- | --- |
| A | 6 | 4 | 10 | Peu de tickets, mais deux gros (LiDAR, SLAM) et des tickets partagés difficiles |
| B | 12 | 1 | 13 | Beaucoup de petits tickets d'interface ; la vue 3D est le plus risqué |
| C | 10 | 3 | 13 | Chemin critique : sessions et proxy bloquent B et D |
| D | 6 | 3 | 9 | Moins de tickets, mais le contenu des leçons est long à écrire et à vérifier |

Si un déséquilibre se confirme en semaine 4, déplacer en priorité : le terminal web côté image de A vers D, et les tableaux de bord de B vers D.

### 22.6 Règles de coordination

1. Un blocage de plus d'un jour ouvre un message à la personne dont on dépend, avec le ticket lié.
2. On ne change jamais un contrat sans prévenir ses consommateurs ; la Merge Request les cite.
3. Chacun montre son travail en démonstration commune toutes les deux semaines, branché sur celui des autres.
4. Quand quelqu'un termine tôt, il aide d'abord son binôme de revue, pas une nouvelle fonction.

## 23. Dossier de l'étudiant A : simulateur et ROS 2

### 23.1 Mission

Tu construis le robot virtuel. À la fin, un programme ROS 2 écrit par un étudiant publie `/cmd_vel` et voit le robot bouger, mesurer et se repérer dans un monde 3D, exactement comme sur un vrai robot. Tout ce que voit B dans le navigateur, et tout ce que lance C dans un conteneur, vient de ton travail.

### 23.2 Périmètre

| Tu possèdes | Chemin |
| --- | --- |
| Simulateur | `runtime/ros2_ws/src/robot_edu_sim/` |
| Robot (URDF) | `runtime/ros2_ws/src/robot_edu_description/` |
| Mondes et leur schéma | `runtime/ros2_ws/src/robot_edu_worlds/` |
| Lancement et paramètres | `runtime/ros2_ws/src/robot_edu_bringup/` |
| Outils de vérification | `runtime/ros2_ws/src/robot_edu_eval/` (et `services/grading/` avec C) |
| Image du conteneur | `runtime/Dockerfile`, `runtime/entrypoint.sh` |

Tu ne touches pas : l'API, le proxy, le front, le Compose général. Tu demandes à leur propriétaire.

### 23.3 Ce qu'il faut apprendre ou réviser

- Les bases de ROS 2 : nodes, topics, `launch`, paramètres, `tf2`, `use_sim_time`.
- PyBullet : `connect(DIRECT)`, `loadURDF`, `setJointMotorControl2`, `stepSimulation`, `rayTestBatch`.
- Les messages `Twist`, `Odometry`, `LaserScan`, `JointState`, `Clock`.
- Le SLAM avec `slam_toolbox` et ses paramètres de base.
- Docker pour construire une image ROS 2 propre.

### 23.4 Tâches pas à pas

**Phase 1 — semaines 1 et 2 (tickets T03, T04, T02)**

1. Faire tourner ROS 2 Jazzy dans Docker : `talker` et `listener` du paquet de démonstration, `ros2 topic echo /chatter`.
2. Installer PyBullet dans l'image (`pip install pybullet`) et exécuter le test du chapitre 8. Si l'installation échoue sur Python 3.12, noter l'erreur et compiler avec `build-essential` ; prévenir l'équipe le jour même.
3. Créer le paquet `robot_edu_sim` (`ros2 pkg create --build-type ament_python robot_edu_sim`).
4. Écrire un URDF minimal de `diffbot-v1` : base, deux roues, une roulette, un lien `laser`.
5. Écrire `sim_node.py` minimal : charge l'URDF, avance la physique, lit `/cmd_vel`, publie `/odom`.
6. Vérifier à la main : `ros2 topic pub /cmd_vel …` fait avancer le robot ; `ros2 topic echo /odom` montre x qui augmente.
7. Enregistrer des exemples de messages dans `docs/contracts.md` pour B et C.

*Fini quand* : `ros2 topic pub` fait avancer le robot de 1 m en 5 s à 0,2 m/s, et B peut s'abonner à `/odom` depuis le navigateur.

```python
# robot_edu_sim/sim_node.py — squelette de départ
import math, rclpy, pybullet as p
from rclpy.node import Node
from geometry_msgs.msg import Twist
from nav_msgs.msg import Odometry
from rosgraph_msgs.msg import Clock

WHEEL_RADIUS, TRACK = 0.05, 0.20            # à aligner avec l'URDF

class SimNode(Node):
    def __init__(self):
        super().__init__('sim_node')
        self.declare_parameter('physics_hz', 240)
        self.declare_parameter('loop_hz', 30)
        self.declare_parameter('urdf', '')
        self.physics_hz = self.get_parameter('physics_hz').value
        loop_hz = self.get_parameter('loop_hz').value
        p.connect(p.DIRECT)
        p.setTimeStep(1.0 / self.physics_hz)
        p.setGravity(0, 0, -9.81)
        p.loadURDF('plane.urdf')            # remplacé par le monde JSON à la phase 2
        self.robot = p.loadURDF(self.get_parameter('urdf').value, [0, 0, 0.1])
        self.t = 0.0
        self.cmd, self.cmd_t = (0.0, 0.0), -1.0
        self.create_subscription(Twist, '/cmd_vel', self.on_cmd, 10)
        self.odom_pub = self.create_publisher(Odometry, '/odom', 10)
        self.clock_pub = self.create_publisher(Clock, '/clock', 10)
        self.steps = self.physics_hz // loop_hz
        self.create_timer(1.0 / loop_hz, self.tick)

    def on_cmd(self, m: Twist):
        self.cmd, self.cmd_t = (m.linear.x, m.angular.z), self.t

    def tick(self):
        v, w = self.cmd if self.t - self.cmd_t < 0.5 else (0.0, 0.0)   # sécurité
        left = (v - w * TRACK / 2) / WHEEL_RADIUS
        right = (v + w * TRACK / 2) / WHEEL_RADIUS
        # appliquer left/right avec p.setJointMotorControl2(..., p.VELOCITY_CONTROL, ...)
        for _ in range(self.steps):
            p.stepSimulation(); self.t += 1.0 / self.physics_hz
        pos, orn = p.getBasePositionAndOrientation(self.robot)
        # remplir et publier Odometry (frame_id='odom', child_frame_id='base_link')
        # publier Clock avec self.t

def main():
    rclpy.init(); rclpy.spin(SimNode())
```

**Phase 2 — semaines 3 et 4 (T11, T12, T16)**

1. Avec D, figer le format du monde JSON (section 13.4) et son schéma `world.schema.json`.
2. Écrire `world_loader.py` : lit le JSON, crée les murs, boîtes et cylindres comme formes de collision PyBullet ; placer le robot au point `spawn`.
3. Publier `/tf` : `odom → base_link` depuis l'état de la base ; `base_link → laser` fixe.
4. Écrire `lidar.py` avec `rayTestBatch` (section 13.6) ; publier `LaserScan` à 10 Hz avec les bons `angle_min`, `angle_increment`, `range_min`, `range_max`.
5. Publier `/clock` et faire fonctionner un node de test avec `use_sim_time:=true`.
6. Ajouter le fichier de paramètres YAML et le bruit gaussien optionnel (graine fixée).
7. Mesurer le CPU d'une session (`docker stats`) et noter la valeur dans `docs/runbooks/`.
8. Enregistrer des exemples de `/scan` et de `/tf` pour B.

*Fini quand* : dans `simple-room`, la distance d'un mur connu est correcte à 1 cm sans bruit, et le robot ne traverse pas les murs.

**Phase 3 — semaines 5 et 6 (T20)**

1. Écrire `runtime/Dockerfile` et `entrypoint.sh` (section 16.1).
2. Lancer `rosbridge_websocket` dans le conteneur, vérifier la connexion depuis un petit script.
3. Exposer un contrôle de santé (`/healthz` sur le port 8081) qui répond seulement quand `sim_node` publie `/odom`.
4. Lire les variables `WORLD`, `ROBOT`, `SESSION_ID`, `ROS_DOMAIN_ID` passées par le gestionnaire de C.
5. Documenter dans `runtime/README.md` : ports, variables, utilisateur, limites attendues.

*Fini quand* : `docker run` de l'image avec les variables démarre un robot visible depuis un client rosbridge en moins de 15 s, en non-root.

**Phase 4 — semaines 7 et 8 (T26 avec C et B)**

1. Ajouter `robot_state_publisher` et publier `/joint_states` pour animer les roues.
2. Ajouter `ttyd` (terminal) dans l'image et le lancer sur le port 7681 dans le dossier `/home/student`.
3. Gérer l'arrêt propre : sur `SIGTERM`, arrêter les nodes et vider les fichiers de l'étudiant.
4. Copier le code de départ dans `/home/student` à l'entrée du conteneur.

**Phase 5 — semaines 9 et 10 (T31, T34)**

1. Configurer `slam_toolbox` (fichier YAML : `use_sim_time`, `scan_topic`, `odom_frame`, `base_frame`, résolution).
2. Cartographier `simple-room` en pilotant le robot ; vérifier que `/map` est publiée et que `map → odom` est cohérent.
3. Écrire `robot_edu_eval` : `pose_probe` (renvoie la pose réelle du robot) et `topic_recorder` (enregistre un topic pendant N secondes).
4. Avec C, brancher ces outils dans `services/grading/`.

**Phase 6 — semaines 11 et 12 (T39)**

1. Écrire les corrigés de référence de chaque leçon dans `robot_edu_examples` et vérifier qu'ils obtiennent le score maximal.
2. Implémenter la comparaison de cartes (IoU) pour `map_compare`.
3. Optimiser : réduire le nombre de rayons ou la cadence, baisser `physics_hz` à 120 si le comportement le permet, profiler `sim_node`.
4. Optionnel : LiDAR 3D (`PointCloud2`) à faible cadence.

**Phases 7 et 8 — semaines 13 à 16**

Corriger les défauts de la recette, documenter le simulateur (limites, paramètres, ajout d'un monde ou d'un robot), vérifier la reproductibilité (même graine, même trajectoire) et répéter la démo.

### 23.5 Ce que tu fournis aux autres

- À B : topics et exemples de messages, URDF, format du monde, comportement des frames.
- À C : l'image `runtime`, ses ports et ses variables, le contrôle de santé.
- À D : les corrigés, la sonde de pose, les paramètres d'exercice.

### 23.6 Ce que tu attends des autres

- De D : le schéma de monde validé en CI, le Compose pour tester l'image.
- De C : les contraintes du gestionnaire (limites, signaux d'arrêt).
- De B : le signalement de tout rendu faux (repère inversé, angle décalé) avec un enregistrement.

### 23.7 Tests que tu écris

Voir 18.2, bloc « Simulateur » : avance de 1 m, rotation de 180°, distance à un mur, déterminisme, arrêt sans commande, cohérence de `LaserScan`.

### 23.8 Pièges fréquents

- Oublier `use_sim_time:=true` dans les nodes de test : les horodatages ne correspondent plus.
- Publier `/tf` et `/odom` avec des horodatages différents : le SLAM rejette les données.
- Utiliser l'horloge murale dans le simulateur : la reproductibilité disparaît.
- Un `frame_id` vide ou mal orthographié : B n'affiche plus rien et l'erreur est difficile à voir.
- Des maillages avec chemins `package://` dans l'URDF : le navigateur ne les trouve pas ; rester sur des formes simples au MVP.
- Lancer trop de rayons à haute cadence : le CPU explose ; mesurer avant d'augmenter.

## 24. Dossier de l'étudiant B : interface web et vue 3D

### 24.1 Mission

Tu construis tout ce que voit l'utilisateur : le site, la page de leçon et surtout la vue 3D qui affiche le robot, le monde et les mesures en direct. Si ton travail est bon, l'étudiant a l'impression de piloter un vrai robot dans une fenêtre, avec son code à côté.

### 24.2 Périmètre

| Tu possèdes | Chemin |
| --- | --- |
| Application React | `web/src/app/`, `web/src/shared/` |
| Pages et fonctions | `web/src/features/` : `auth`, `courses`, `lesson`, `session`, `editor`, `terminal`, `viewer3d`, `ros`, `progress`, `profile` |
| Traductions | `web/public/locales/` (avec D pour les textes) |
| Tests du front | `web/tests/` |

Tu ne touches pas : l'API, le simulateur, l'infrastructure. Si tu as besoin d'un champ, tu le demandes à C avec l'exemple de réponse attendu.

### 24.3 Ce qu'il faut apprendre ou réviser

- React et TypeScript : composants, hooks, routes, requêtes avec TanStack Query.
- Three.js : scène, caméra, lumières, géométries, `BufferGeometry`, boucle de rendu, libération des ressources.
- Les repères ROS et la conversion vers Three.js (section 14.4) ; les messages `Odometry`, `LaserScan`, `TFMessage`, `JointState`.
- `roslibjs` pour parler à rosbridge ; `urdf-loader` pour charger un robot.
- Monaco et xterm.js : intégration dans React, redimensionnement.

### 24.4 Tâches pas à pas

**Phase 1 — semaines 1 et 2 (T05, T06, T02)**

1. Créer le projet : `npm create vite@latest web -- --template react-ts`, ajouter `three`, `roslib`, `zustand`, `@tanstack/react-query`, `react-router-dom`.
2. Afficher une scène Three.js avec sol, grille, axes et un cube ; contrôler la caméra avec la souris.
3. Déplacer le cube avec des valeurs de test (cercle) pour vérifier la boucle de rendu.
4. Écrire `scene/frames.ts` (section 14.4) et ses tests : `(1,0,0)` → `(1,0,0)` et `(0,1,0)` → `(0,0,-1)`.
5. Écrire `rosClient.ts` et `topicStore.ts` ; se connecter au rosbridge d'A et s'abonner à `/odom`.
6. Remplacer les valeurs de test par la pose réelle ; envoyer la liste des champs d'API dont tu as besoin à C.

*Fini quand* : un cube (puis un robot simple) se déplace dans le navigateur quand A publie `/odom`.

**Phase 2 — semaines 3 et 4 (T13, T14, T17)**

1. `WorldLayer` : construire murs, boîtes, cylindres et objectifs à partir du JSON du monde (section 13.4) ; couleurs et hauteurs cohérentes.
2. `RobotModel` : charger l'URDF avec `urdf-loader`, appliquer la pose de `/tf` et faire tourner les roues avec `/joint_states`.
3. `ScanLayer` : points du `LaserScan` (section 14.6), ignorer les valeurs invalides, tampon réutilisé.
4. Caméras : libre (orbite) et vue de dessus orthographique, bouton pour basculer.
5. Vérifier visuellement : les points du scan tombent sur les murs du monde. Sinon, décrire l'écart à A avec une capture et les valeurs.

*Fini quand* : dans `simple-room`, le scan dessine le contour des murs et le robot tourne et avance comme dans le simulateur.

**Phase 3 — semaines 5 et 6 (T22, T24)**

1. Générer les types TypeScript depuis l'OpenAPI de C (`make types`).
2. Routes et gardes : `/`, `/courses`, `/courses/:slug`, `/login`, `/register`, `/dashboard`.
3. Pages d'authentification avec gestion d'erreur (champs invalides, compte inexistant, trop de tentatives).
4. Catalogue et fiche de cours avec filtres.
5. `useSession` et `sessionStore` : machine d'états de la section 14.7 ; interrogation de `GET /sessions/{id}`.
6. `SessionBar` : état, minuteur, bouton Arrêter, bandeau de reconnexion.

**Phase 4 — semaines 7 et 8 (T25, T26, T28)**

1. `CodeEditor` avec Monaco : fichiers de l'espace de travail, enregistrement automatique (après 1 s sans frappe), indicateur « enregistré ».
2. `TerminalPanel` avec xterm.js : connexion WebSocket du terminal, redimensionnement, reconnexion.
3. `LessonPage` : panneaux redimensionnables (section 9.3), texte du cours rendu en Markdown sûr, indices dévoilables.
4. Brancher la barre basse : liste des topics et cadences, bouton Soumettre.
5. Tester la politique de sécurité (CSP) avec Monaco et Three.js avec D.

*Fini quand* : un utilisateur connecté ouvre la leçon 1, écrit un node dans l'éditeur, le lance dans le terminal et voit le robot bouger.

**Phase 5 — semaines 9 et 10 (T32, T35)**

1. `MapLayer` : `OccupancyGrid` rendue sur un plan texture ; vérifier le sens d'indexation (l'axe vertical est souvent inversé).
2. `TrailLayer` : trajectoire issue de `/odom` (limiter à 5 000 points).
3. Soumission : `POST /attempts`, attente du résultat, affichage du score et des retours par critère.
4. Bouton « Leçon suivante » et mise à jour de la progression.

**Phase 6 — semaines 11 et 12 (T38)**

1. Tableau de bord et page de progression.
2. Badge de source (« Simulation », « Dataset rejoué »), bascule des couches, clic sur un point du scan pour voir distance et angle.
3. Accessibilité : navigation clavier, contrastes, libellés.
4. Traductions françaises et anglaises complètes ; pages d'erreur.

**Phases 7 et 8 — semaines 13 à 16**

Performance (60 images par seconde, chargement différé), tests bout en bout avec D, corrections de la recette et répétition de la démo.

### 24.5 Exemple : branchement de la boucle de rendu

```ts
// viewer3d/scene/SceneManager.ts (extrait)
export class SceneManager {
  private raf = 0;
  constructor(private renderer: WebGLRenderer, private scene: Scene, private camera: Camera,
              private layers: Layer[], private topics: () => TopicSnapshot) {}

  start() {
    const loop = () => {
      if (document.visibilityState === 'visible') {
        const snap = this.topics();                 // dernières valeurs de chaque topic
        for (const layer of this.layers) layer.update(snap);
        this.renderer.render(this.scene, this.camera);
      }
      this.raf = requestAnimationFrame(loop);
    };
    loop();
  }
  dispose() { cancelAnimationFrame(this.raf); this.layers.forEach(l => l.dispose()); this.renderer.dispose(); }
}
```

### 24.6 Ce que tu fournis aux autres

- À C : la liste des champs d'API par page, très tôt, et les retours sur les formats de réponse.
- À A : un rendu qui rend visibles les erreurs de frames et d'horodatage (un cube de test, un badge d'avertissement).
- À D : le format exact de rendu du Markdown (balises, blocs de code, indices) et des scénarios d'interface pour les tests bout en bout.

### 24.7 Ce que tu attends des autres

- De A : exemples de messages et enregistrements de `/odom`, `/scan`, `/tf` dès la semaine 2 à 4.
- De C : OpenAPI, types, endpoint de session simulé (semaine 2), connexion réelle (semaine 6).
- De D : cours de démonstration complet (semaine 4), Compose qui marche, textes des leçons.

### 24.8 Tests que tu écris

Voir 18.2, bloc « Front » : conversion de repères, scan invalide, reconnexion, reprise de session. Ajouter des tests de composants pour la barre de session et la machine d'états, et un test bout en bout du parcours E1 à E4 avec D.

### 24.9 Pièges fréquents

- Convertir les repères à plusieurs endroits : un petit écart finit toujours par apparaître ; tout passe par `frames.ts`.
- Redessiner React à chaque message ROS : la page rame ; les messages vont dans un store, le rendu les lit à son rythme.
- Créer une nouvelle géométrie à chaque scan : fuite mémoire ; réutiliser les tampons et appeler `dispose`.
- Oublier de se désabonner en quittant la page : les messages continuent et la mémoire grossit.
- Stocker un jeton dans `localStorage` : risque inutile ; utiliser les cookies `HttpOnly` pour l'accès et garder le jeton de session en mémoire.
- Charger Monaco et Three.js dès l'accueil : le site devient lent ; import dynamique sur la page de leçon.

## 25. Dossier de l'étudiant C : API, sessions et sécurité

### 25.1 Mission

Tu construis le cœur serveur : les comptes, le catalogue, la progression, et surtout ce qui démarre, relie et arrête les conteneurs de simulation en toute sécurité. Ton travail est sur le chemin critique : B ne peut pas brancher le vrai robot avant ton proxy, et D ne peut pas tester sans ton gestionnaire de sessions.

### 25.2 Périmètre

| Tu possèdes | Chemin |
| --- | --- |
| API (monolithe modulaire) | `services/api/` : modules `auth`, `users`, `catalog`, `content`, `workspace`, `sessions`, `progress`, `artifacts` |
| Gestionnaire de sessions | `services/session-manager/` |
| Proxy temps réel | `services/realtime-proxy/` |
| Modèle de données et migrations | `services/api/migrations/` (schémas du chapitre 11) |
| Vérification (avec A) | `services/grading/` |
| Contrats d'API et événements | chapitre 12, `docs/contracts.md`, `docs/api/openapi.json` |

Tu ne touches pas : le code du simulateur, le front, la CI générale (demande à D).

### 25.3 Ce qu'il faut apprendre ou réviser

- Spring Boot, Spring Data JPA (Hibernate), Bean Validation, Flyway.
- PostgreSQL : transactions, verrous de ligne, index.
- Authentification : Argon2id, JWT, cookies `HttpOnly`, rafraîchissement de jeton.
- Bibliothèque docker-java : création de conteneurs avec limites, réseaux, étiquettes.
- Spring WebSocket avec `Java-WebSocket` ; le protocole rosbridge (JSON) pour filtrer les messages.
- Redis Streams : groupes de consommateurs, acquittement, déduplication.

### 25.4 Tâches pas à pas

**Phase 1 — semaines 1 et 2 (T07, T08)**

1. Créer `services/api` : `pyproject.toml`, `app/main.py`, route `GET /health`.
2. Ajouter PostgreSQL, la configuration (`application.yml`), Spring Data JPA et Flyway ; première migration (schémas et tables `auth` et `users`).
3. Implémenter `POST /sessions` en version simulée : renvoie une session `running` avec un bloc `connection` fictif pointant sur le rosbridge de développement.
4. Publier le schéma OpenAPI (`docs/api/openapi.json`) pour que B génère ses types.
5. Recevoir de B la liste des champs d'API par page et ajuster le contrat.

*Fini quand* : B appelle `POST /sessions` depuis le navigateur et reçoit une réponse conforme à la section 12.4.

**Phase 2 — semaines 3 et 4 (T15, T18)**

1. Module `auth` : `register`, `login`, `refresh`, `logout` ; hachage Argon2id ; cookies `HttpOnly`, `Secure`, `SameSite=Lax` ; verrouillage après 5 échecs.
2. Module `users` : profil, langue, thème, quota individuel.
3. Filtre JWT Spring Security et paramètre `currentUser` disponible dans toutes les routes protégées.
4. Module `catalog` : tables du chapitre 11, routes `GET /courses`, `/courses/{slug}`, `/lessons/{id}`, `/worlds`, `/robots`.
5. Module `content` : lit `content/courses/**` (YAML et Markdown), nettoie le Markdown et sert le texte de chaque leçon.
6. Script `seed` : crée le cours d'exemple, les mondes, le robot, un compte de test.
7. Donner à D les comptes de test (semaine 3) et le format de `lesson.yaml` validé ensemble.

**Phase 3 — semaines 5 et 6 (T21, T23)**

1. `session-manager` : API interne (`POST/GET/DELETE /internal/sessions`, `GET /internal/capacity`) ; création d'un conteneur avec toutes les limites de la section 16.3 et un réseau par session.
2. Attacher le proxy temps réel au réseau de la session ; attendre le contrôle de santé de l'image d'A avant de passer la session à `running`.
3. `realtime-proxy` : accepter une connexion WebSocket, lire le premier message `auth`, vérifier le jeton (signature, expiration, session, utilisateur), router vers rosbridge ou le terminal.
4. Filtre rosbridge : liste blanche d'opérations, de topics et de types ; plafond de fréquence et de taille ; plafond de vitesse sur `/cmd_vel`.
5. Brancher l'API : `POST /sessions` crée la ligne en base, publie `session.requested` et le gestionnaire la traite ; `GET /sessions/{id}` renvoie le bloc `connection` réel.
6. Donner à B le vrai bloc de connexion (semaine 6).

*Fini quand* : une session créée par l'API démarre un vrai conteneur, le navigateur se connecte par le proxy, et un second utilisateur ne peut pas utiliser le jeton du premier.

```java
// controller/SessionController.java (extrait)
@RestController
@RequestMapping("/api/v1/sessions")
public class SessionController {
    private final SessionService sessionService;

    @PostMapping
    @ResponseStatus(HttpStatus.ACCEPTED)
    public SessionDto create(@Valid @RequestBody SessionCreateRequest body,
                             @RequestHeader("Idempotency-Key") String idempotencyKey,
                             @AuthenticationPrincipal AppUser user) {
        return sessionService.create(user, body, idempotencyKey);
    }
}

// service/SessionService.java (extrait)
@Transactional
public SessionDto create(AppUser user, SessionCreateRequest body, String key) {
    profileRepository.lockByUserId(user.getId());            // SELECT ... FOR UPDATE
    return sessionRepository.findByUserIdAndIdempotencyKey(user.getId(), key)
        .map(SessionMapper::toDto)                           // même clé, même résultat
        .orElseGet(() -> {
            if (sessionRepository.countActive(user.getId()) >= user.getMaxSessions())
                throw new ApiException("session_quota_exceeded", HttpStatus.TOO_MANY_REQUESTS);
            Session s = new Session(user, SessionState.REQUESTED, key,
                                    Instant.now().plus(props.getSessionTtl()), resolveExercise(body));
            sessionRepository.save(s);
            events.publish("session.requested", Map.of("sessionId", s.getId().toString()));
            return SessionMapper.toDto(s);
        });
}
```

**Phase 4 — semaines 7 et 8 (T27, T29, T26)**

1. Module `workspace` : lecture et écriture de fichiers par leçon, limites de taille, chemin sans `..`.
2. Gestionnaire : copier les fichiers de l'étudiant dans le conteneur au démarrage et les récupérer à l'arrêt.
3. Expiration : arrêt automatique à `expires_at`, événement `session.expired`, fermeture des WebSockets par le proxy.
4. Réconciliation toutes les 30 s : conteneurs sans session valide supprimés ; sessions sans conteneur marquées en échec.
5. Proxy du terminal : messages `input`, `resize`, `output`, `exit` ; taille maximale 64 Ko.

**Phase 5 — semaines 9 et 10 (T33, T34)**

1. Module `progress` : `POST /attempts` (idempotent), `GET /attempts/{id}`, `GET /progress/me`.
2. Publier `attempt.submitted` ; avec A, écrire `services/grading/worker.py` qui lit l'événement, appelle la sonde dans le conteneur, calcule le score selon le barème YAML et publie `grading.completed`.
3. Recalculer `lesson_progress` à chaque résultat.

**Phase 6 — semaines 11 et 12 (T37)**

1. Quotas globaux et par utilisateur ; réponse `503 capacity_full` avec `Retry-After` ; file d'attente simple si le temps le permet.
2. Prolongation de session (`POST /sessions/{id}/extend`) avec limite.
3. Export des données du compte et suppression (`DELETE /users/me`).

**Phases 7 à 8 — semaines 13 à 16 (T40)**

Durcissement avec la liste 17.5, rétention et nettoyage (11.4), alertes de capacité, sauvegarde de la base, corrections de la recette, version `v1.0`.

### 25.5 Ce que tu fournis aux autres

- À B : OpenAPI et types (semaine 2), endpoint de session simulé (semaine 2), connexion réelle (semaine 6), résultats de tentative.
- À A : variables, ports, limites et signaux d'arrêt que le gestionnaire applique à l'image.
- À D : comptes de test, `seed`, API stable pour les tests d'intégration, format de barème.

### 25.6 Ce que tu attends des autres

- De A : l'image `runtime` avec ports, variables et contrôle de santé (semaine 5), les sondes de vérification (semaine 10).
- De B : les besoins d'API de chaque page, des retours rapides sur les formats.
- De D : le Compose, la CI, les scénarios de test et les leçons du catalogue.

### 25.7 Tests que tu écris

Voir 18.2, blocs « API », « Gestionnaire de sessions » et « Proxy temps réel » : propriété des sessions, idempotence, quotas en parallèle, nettoyage des orphelins, jeton invalide, opération rosbridge interdite, message trop gros.

### 25.8 Pièges fréquents

- Monter le socket Docker dans un conteneur exposé : à éviter absolument (voir 16.2).
- Vérifier les droits seulement dans la route : vérifier aussi dans le service, car les événements ne passent pas par les routes.
- Compter les sessions actives sans verrou : deux requêtes simultanées dépassent le quota.
- Passer une session à `running` avant que le simulateur soit prêt : B se connecte trop tôt ; attendre le contrôle de santé.
- Oublier de supprimer le réseau Docker d'une session expirée : fuite de ressources.
- Journaliser un jeton, un mot de passe ou le code d'un étudiant : jamais.
- Faire confiance au score envoyé par le client : le score vient uniquement de `grading`.

## 26. Dossier de l'étudiant D : contenu, infrastructure et qualité

### 26.1 Mission

Tu es la personne qui fait tenir l'ensemble : tu écris les leçons que les étudiants suivront, tu fais tourner toute la plateforme avec une seule commande, tu automatises les vérifications et tu prouves que ça marche. Sans ton travail, le projet est un assemblage de morceaux qui ne se lancent pas ensemble et qui n'enseignent rien.

### 26.2 Périmètre

| Tu possèdes | Chemin |
| --- | --- |
| Cours, leçons, code de départ, barèmes | `content/courses/`, `services/grading/app/rubrics/` |
| Datasets et manifestes | `data/` |
| Compose et déploiement | `compose*.yaml`, `infra/`, `.env.example`, `Makefile` |
| CI GitLab | `.gitlab-ci.yml`, `.gitlab/` |
| Tests d'intégration, bout en bout, charge | `tests/` |
| Outils de contrôle | `tools/` (validation des mondes et des leçons) |
| Documentation et contrats | `docs/` (avec tous pour `contracts.md` et `frames.md`) |

Tu ne touches pas le code métier de A, B ou C ; tu leur envoies un ticket précis quand un test révèle un problème.

### 26.3 Ce qu'il faut apprendre ou réviser

- Docker et Docker Compose : réseaux, volumes, `healthcheck`, `depends_on`, profils.
- GitLab CI : `stages`, `rules`, services, artefacts, variables protégées.
- Caddy ou Nginx : reverse proxy, TLS, en-têtes de sécurité.
- Playwright (tests de navigateur), `JUnit 5`, Locust ou k6 (charge).
- Pédagogie : rédiger une leçon avec un objectif mesurable, des indices et un retour utile.
- Les bases de ROS 2 et du LiDAR (assez pour écrire et vérifier les leçons).

### 26.4 Tâches pas à pas

**Phase 1 — semaines 1 et 2 (T01, T09, T02, T10)**

1. Créer le projet GitLab, invités, `main` protégée, `CODEOWNERS` selon la section 15.2, modèles de tickets et de Merge Request.
2. Créer l'arborescence du chapitre 15 avec un `README.md` par dossier.
3. Écrire le `Makefile` (`up`, `down`, `test`, `lint`, `seed`, `types`) et `.env.example`.
4. Écrire `compose.yaml` avec `db`, `redis`, `api`, `web` et un conteneur `runtime-image` de test ; vérifier `docker compose up` depuis un clone propre.
5. Configurer une CI minimale : lint Java et TypeScript.
6. Organiser la rédaction de `contracts.md` et `frames.md` avec les trois autres.
7. Écrire la procédure de vérification de la tranche verticale et la faire exécuter par quelqu'un d'autre.

*Fini quand* : un nouveau membre clone le dépôt, lance `make up` et voit la page d'accueil, et la tranche verticale de la semaine 2 est reproductible.

**Phase 2 — semaines 3 et 4 (T11, T19)**

1. Avec A, figer le schéma JSON du monde et écrire `tools/validate_worlds.py` ; l'ajouter à la CI.
2. Définir avec C le format de `lesson.yaml` ; écrire `tools/lint_content.py` qui vérifie métadonnées, liens et fichiers de départ.
3. Rédiger les leçons 1 à 3 (section 19.2) : texte français, code de départ, critères de réussite, indices.
4. Fournir à B un cours de démonstration complet (semaine 4) pour tester l'interface.
5. Écrire le premier barème et la fiche de vérification dans `docs/exercises/`.
6. Première mesure de charge : relevé du CPU et de la mémoire d'une session (avec A).

**Phase 3 — semaines 5 et 6**

1. Ajouter au Compose les réseaux, le proxy et le gestionnaire de sessions de C ; préparer le profil `build-only` de l'image `runtime`.
2. Écrire les premiers tests d'intégration (`tests/integration/`) : une session démarre, se connecte et s'arrête ; le conteneur est supprimé.
3. Premier test d'évasion manuel : depuis le terminal d'une session, essayer d'atteindre Internet, un autre conteneur, le socket Docker ; documenter le résultat.
4. Mesurer la latence entre commande et mouvement ; noter la méthode.

**Phase 4 — semaines 7 et 8 (T30)**

1. Rédiger les leçons 4 à 6.
2. Tester la politique de sécurité du site (CSP) avec Monaco et Three.js, avec B.
3. Installer Playwright et automatiser les scénarios E1 à E3 ; les ajouter à la CI de nuit.
4. Ajouter le job de test du paquet ROS (`colcon test`) à la CI.

**Phase 5 — semaines 9 et 10 (T36)**

1. Rédiger les mondes 2 et 3 (couloir, labyrinthe) en JSON avec A ; les valider en CI.
2. Écrire les barèmes des leçons et les vérifications à la main avec les corrigés d'A.
3. Automatiser le scénario E4 (soumission et score).
4. Préparer les manifestes de datasets et vérifier la licence de chaque jeu avant ajout.

**Phase 6 — semaines 11 et 12 (T39)**

1. Rédiger la leçon de SLAM (leçon 13) avec A et la tester de bout en bout.
2. Automatiser E5 à E8 (reprise de session, expiration, accès refusé, capacité pleine).
3. **Test de charge** avec Locust ou k6 : monter par paliers (2, 4, 8, 12 sessions), mesurer CPU, mémoire, latence ; fixer `MAX_CONCURRENT_SESSIONS` avec C.
4. Geler les fonctionnalités à la fin de la semaine 12 et le dire à tout le monde.

**Phases 7 à 8 — semaines 13 à 16 (T40, T41, T42)**

1. Passer la liste 17.5 avec C ; audit des dépendances en CI.
2. Mettre en place les métriques et les alertes de base (section 16.8).
3. Exécuter la recette E1 à E10 et ouvrir un ticket par défaut avec le propriétaire du composant.
4. Écrire le manuel d'installation, le rapport et le scénario de démonstration ; préparer la vidéo de secours ; restaurer une sauvegarde de la base.
5. Étiqueter `v1.0` avec C.

### 26.5 Exemple de leçon à écrire (squelette)

```markdown
# Lire un scan LiDAR

**Objectif** : afficher la distance de l'obstacle situé devant le robot.
**Durée** : 45 minutes · **Prérequis** : leçon 2

## 1. Ce que mesure un LiDAR
(3 à 5 phrases, un schéma)

## 2. Le message `LaserScan`
Tableau : `angle_min`, `angle_increment`, `ranges`.

## 3. À vous
1. Ouvrez `scan_listener.py`.
2. Complétez la fonction `distance_devant(scan)`.
3. Lancez : `ros2 run robot_edu_examples scan_listener`.

> **Indice 1** : l'indice du rayon de face est `len(ranges) // 2`.
> **Indice 2** : ignorez les valeurs `inf`.

## 4. Critères de réussite
- Votre node s'abonne à `/scan`.
- La distance affichée est à 2 cm de la valeur attendue.
```

### 26.6 Ce que tu fournis aux autres

- À tous : Compose, `make up`, CI, `contracts.md` à jour, recette automatisée.
- À A : schéma de monde validé en CI, tests d'intégration de l'image.
- À B : un cours de démonstration, le format du Markdown, des scénarios bout en bout.
- À C : comptes et données de départ, barèmes, tests d'intégration et de charge.

### 26.7 Ce que tu attends des autres

- De A : monde d'exemple, paramètres d'exercice, corrigés de référence.
- De B : textes d'interface à traduire, indications sur le rendu du Markdown.
- De C : `seed`, bloc de connexion réel, API stable, valeurs de quotas.

### 26.8 Tests que tu écris

Voir 18.1 et 18.3 : intégration (session réelle), bout en bout E1 à E10, charge, validation des mondes et des leçons, liste de sécurité 17.5. Tu mets aussi en place l'indicateur de succès du projet : la recette E1 à E10 passe depuis une installation propre.

### 26.9 Pièges fréquents

- Laisser le Compose fonctionner uniquement sur ta machine : tester chaque semaine depuis un clone neuf.
- Écrire les leçons après le reste : elles révèlent des problèmes de simulateur ou d'interface, donc elles doivent être écrites tôt, quitte à être simples.
- Des barèmes trop stricts (tolérance de 1 cm) sur une simulation avec du bruit : tester avec les corrigés et ajuster.
- Ajouter un dataset sans vérifier sa licence : ne jamais l'ajouter au dépôt sans manifeste complet.
- Laisser les tests de charge pour la dernière semaine : si la capacité est trop faible, il n'y a plus de temps pour corriger.
- Mettre des secrets dans la CI ou dans Git : variables protégées de GitLab uniquement.

### 26.10 Liste de contrôle hebdomadaire commune aux quatre dossiers

- [ ] Mes tickets de la semaine sont à jour dans GitLab (en cours, bloqué, terminé).
- [ ] Mes Merge Requests ouvertes ont un relecteur et un pipeline vert.
- [ ] Les contrats que j'ai touchés sont à jour dans `docs/contracts.md`.
- [ ] J'ai prévenu ceux qui dépendent de mes livraisons de la semaine suivante.
- [ ] Un collègue a pu lancer mon travail en suivant mon `README`.

# Partie 4 — Schémas récapitulatifs et langages

## 27. Schémas récapitulatifs : qui fait quoi, qui parle à qui

Ce chapitre résume tout ce qui précède en trois vues. Si tu découvres le projet, commence ici : la carte globale (27.1) montre toutes les briques et leur rôle, la séquence (27.2) montre ce qui se passe quand un étudiant clique sur « Lancer », et le tableau des connexions (27.3) donne les protocoles et les ports. Pour le détail de chaque brique, les chapitres 2, 10 et 16 restent la référence.

### 27.1 La carte globale

&#91;embedded content: architecture globale · 4 étages, 9 briques\]

La carte se lit de haut en bas. Le navigateur est le seul élément que l'étudiant voit ; tout ce qu'il envoie passe par la passerelle Caddy, qui est la seule porte exposée. Derrière elle, trois services ont chacun un métier : l'API gère les données du site, le proxy temps réel garde l'accès au robot, et la gestion des sessions fabrique et détruit les conteneurs. Le conteneur de session, en couleur, est l'endroit où tourne le code de l'étudiant : c'est le seul endroit où du code non fiable s'exécute, donc le plus durci.

| Brique | Rôle en une phrase | Elle parle à |
| --- | --- | --- |
| Navigateur | Affiche la leçon, la scène 3D, l'éditeur et le terminal ; ne parle jamais directement à Docker, à la base ou à Redis | Passerelle Caddy |
| Passerelle Caddy | Termine le HTTPS et aiguille chaque requête vers le bon service | API, proxy temps réel |
| API (Spring Boot) | Comptes, catalogue, contenu, progression, demande d'ouverture de session | PostgreSQL, Redis, gestion des sessions |
| Proxy temps réel | Contrôle le jeton de session, filtre les messages rosbridge, relaie le terminal | Conteneur de session |
| Gestion des sessions | Seul service autorisé à parler à Docker : crée, surveille et arrête les conteneurs | Docker, Redis |
| PostgreSQL | Mémoire durable du site | API, worker de notation |
| Redis | Bus d'événements entre les services | API, sessions, notation |
| Worker de notation | Note les soumissions en dehors de l'API pour ne pas la bloquer | Redis, PostgreSQL |
| Conteneur de session | Un monde simulé, un robot et un terminal par étudiant | Proxy temps réel |

La propriété de chaque brique par les quatre étudiants est dans la matrice du chapitre 22.

### 27.2 La vie d'une session, échange par échange

&#91;embedded content: vie d'une session · 11 échanges, 5 acteurs\]

Les échanges 1 à 4 ouvrent la session : l'API ne crée rien elle-même, elle délègue à la gestion des sessions, qui est la seule à toucher Docker, puis renvoie au navigateur un jeton court et l'adresse à utiliser. Les échanges 5 à 8 sont la session ouverte : le navigateur se connecte au proxy avec ce jeton, envoie des commandes au robot et reçoit en continu les mesures qui alimentent la scène Three.js. Les échanges 9 à 11 sont la fin : le code est soumis, la notation se fait ensuite en arrière-plan par le worker via Redis, et le conteneur est supprimé.

### 27.3 Qui parle à qui : protocoles et ports

| Connexion | Protocole | Détail |
| --- | --- | --- |
| Navigateur → Caddy | HTTPS, WebSocket | Seul point d'entrée exposé ; cookie JWT pour l'API, jeton court par session pour le WebSocket |
| Proxy → rosbridge | WebSocket JSON | Port 9090 dans le conteneur ; seules les opérations et les topics de la liste blanche passent |
| Proxy → ttyd | WebSocket | Port 7681 ; terminal du conteneur |
| Gestion des sessions → Docker | API Docker | Seul service à y accéder ; un réseau interne par session |
| API → PostgreSQL | SQL | Un schéma par capacité, migrations Flyway |
| Services → Redis | Redis Streams | Événements entre API, sessions et notation |
| Contrôles de santé | HTTP | Port 8081 sur les services |

Règle à afficher dans l'équipe : le navigateur ne parle jamais directement à rosbridge, à ttyd, à Docker, à PostgreSQL ou à Redis. Si une fonctionnalité a l'air de demander cela, c'est qu'un service intermédiaire manque.

# Partie 4 — Livrables UML et Spring Boot exigés par la grille

Cette partie produit, dans l'ordre de la consigne du cours, les livrables notés : analyse, diagrammes UML, code Spring Boot, tests et documentation. Elle part de l'application métier de la plateforme (comptes, cours, exercices, sessions, tentatives, scores) ; le simulateur ROS 2 n'y figure que comme service appelé.

## 4.1 Analyse fonctionnelle (critère 1, 5 points)

La plateforme a trois acteurs humains et un système externe ; les besoins métier se résument à apprendre la robotique en pratiquant dans un simulateur et à être noté automatiquement.

| Acteur | Rôle | Besoin principal |
| --- | --- | --- |
| Visiteur | personne non connectée | voir le catalogue, créer un compte |
| Étudiant | apprenant connecté | suivre un cours, coder, voir le robot réagir, obtenir un score |
| Administrateur | responsable du contenu | créer et maintenir cours, leçons et exercices, gérer les comptes |
| Simulateur ROS 2 (système) | conteneur d'une session | fournir les mesures du robot et la pose réelle pour la notation |

**Fiche de spécifications fonctionnelles.** Les règles de gestion sont des propositions à valider avec le client avant le CdC initial du 01/11.

| Id | Acteur | Fonctionnalité | Règle de gestion proposée |
| --- | --- | --- | --- |
| F01 | Visiteur | Créer un compte | e-mail unique ; mot de passe stocké haché |
| F02 | Visiteur, Étudiant | Se connecter | un jeton d'accès à durée courte ; réponse 401 si identifiants faux |
| F03 | Visiteur | Consulter le catalogue | seuls les cours publiés ; liste paginée |
| F04 | Étudiant | S'inscrire à un cours | une seule inscription par cours |
| F05 | Étudiant | Consulter une leçon | réservée aux inscrits |
| F06 | Étudiant | Démarrer une session de simulation | un nombre maximal de sessions actives par étudiant ; même clé d'idempotence, même résultat |
| F07 | Étudiant | Enregistrer un fichier de la session | taille maximale ; fichier lié à l'étudiant et à l'exercice |
| F08 | Étudiant | Soumettre une tentative | session active obligatoire ; le score est calculé après la soumission |
| F09 | Étudiant | Consulter ses scores et sa progression | le meilleur score par exercice est conservé |
| F10 | Étudiant | Arrêter une session | les ressources sont libérées ; arrêt automatique à expiration |
| F11 | Administrateur | Créer, modifier, supprimer cours, leçons, exercices | suppression refusée s'il existe des tentatives (on dépublie) |
| F12 | Administrateur | Définir le barème d'un exercice | barème versionné, appliqué à la tentative au moment de la soumission |
| F13 | Administrateur | Désactiver un compte | les sessions actives du compte sont arrêtées |

Livrable de l'étape 1 de la consigne : cette fiche, exportée en PDF dans `docs/rendus/` et jointe au CdC.

## 4.2 Diagramme de cas d'utilisation (critère 2, 5 points)

Les 13 cas correspondent un à un aux fonctionnalités F01 à F13 de la fiche 4.1 : le visiteur ouvre l'accès, l'étudiant suit un cours et lance des sessions, l'administrateur gère le contenu et les comptes.

&#91;embedded content: cas d'utilisation · 4 acteurs, 13 cas\]

| Cas d'utilisation | Fonctionnalité | Acteur |
| --- | --- | --- |
| Consulter le catalogue | F03 | Visiteur |
| Créer un compte | F01 | Visiteur |
| Se connecter | F02 | Visiteur, étudiant, administrateur |
| S'inscrire à un cours, consulter une leçon | F04, F05 | Étudiant |
| Démarrer, arrêter une session | F06, F10 | Étudiant (le simulateur fournit le conteneur) |
| Enregistrer un fichier | F07 | Étudiant |
| Soumettre une tentative, consulter ses scores | F08, F09 | Étudiant (le simulateur fournit la pose) |
| Gérer cours, leçons, exercices ; définir un barème | F11, F12 | Administrateur |
| Désactiver un compte | F13 | Administrateur |

## 4.3 Diagramme de classes et entités JPA (critères 3 et 7, 20 points)

Le diagramme ne contient que les dix entités persistantes de l'application ; chaque classe du schéma devient une classe `@Entity` du package `model`, et chaque trait une annotation de relation. Il est à rapprocher du schéma de données du chapitre 11.

&#91;embedded content: diagramme de classes · 10 entités JPA et leurs associations\]

| Association | Annotation JPA | Clé étrangère |
| --- | --- | --- |
| User ↔ Role | `@ManyToMany` | table `user_roles` |
| User → Enrollment | `@OneToMany(mappedBy = "user")` et `@ManyToOne` | `enrollment.user_id` |
| Enrollment → Course | `@ManyToOne` | `enrollment.course_id` |
| Course → Lesson | `@OneToMany(mappedBy = "course", cascade = ALL, orphanRemoval = true)` et `@ManyToOne` | `lesson.course_id` |
| Lesson → Exercise | `@OneToMany` et `@ManyToOne` | `exercise.lesson_id` |
| User → Session | `@ManyToOne` | `session.user_id` |
| Session → Exercise (facultatif) | `@ManyToOne(optional = true)` | `session.exercise_id`, nul en mode libre |
| Session → Attempt | `@OneToMany` et `@ManyToOne` | `attempt.session_id` |
| Attempt → Exercise | `@ManyToOne` | `attempt.exercise_id` |
| Attempt → Score | `@OneToOne(mappedBy = "attempt")` | `score.attempt_id`, unique |
| User → WorkspaceFile | `@ManyToOne` | `workspace_file.user_id` |
| WorkspaceFile → Exercise | `@ManyToOne` | `workspace_file.exercise_id` |

**Énumérations.** `Role.name` : `STUDENT`, `ADMIN`. `Exercise.type` : `POSE_GOAL`, `TOPIC_SHAPE`, `MAP_COMPARE`, `QUIZ` (les quatre correcteurs du chapitre 15). `Session.state` : l'énumération `SessionState`, dont les valeurs sont celles du chapitre 12.4.

## 4.4 Diagramme de séquence (critère 4, 10 points)

Le schéma 0.2 montre le parcours complet d'une session. Ici on zoome sur une seule requête, la soumission d'une tentative, pour montrer le découpage en couches exigé par la grille.

&#91;embedded content: diagramme de séquence · soumission d'une tentative\]

- **Controller** : lit le JWT, valide le DTO avec `@Valid`, appelle le service, renvoie un `201`. Aucune règle métier.
- **Service** : `@Transactional`, charge l'exercice, choisit le correcteur selon `Exercise.type`, calcule le score et enregistre la tentative avec son score.
- **Repository** : interfaces `JpaRepository`, aucune requête SQL écrite à la main sauf cas particulier.
- **Cas d'erreur** : exercice introuvable → `ResourceNotFoundException` → `404` ; DTO invalide → `400` ; token absent ou expiré → `401`. Tout est transformé en JSON par un `@RestControllerAdvice` (section 4.8).

## 4.5 Diagramme d'activité (critère 5, 10 points)

Le diagramme décrit l'algorithme du service `AttemptService.submit()` : trois décisions, deux sorties d'erreur, un seul chemin de succès. Il prolonge la séquence 4.4 en montrant ce qui se passe à l'intérieur du service.

&#91;embedded content: diagramme d'activité · soumission et notation d'une tentative, 3 décisions\]

| Décision | Condition | Si non |
| --- | --- | --- |
| DTO valide ? | `@NotNull`, `@NotBlank`, taille maximale du code envoyé | `MethodArgumentNotValidException` → `400` |
| Exercice trouvé ? | `exerciseRepository.findById()` non vide | `ResourceNotFoundException` → `404` |
| Score ≥ seuil ? | `score.value >= exercise.passingScore` | statut `FAILED`, la tentative est quand même enregistrée |

Un deuxième diagramme d'activité, plus court, est possible pour la création de session (idempotence, quota d'une session par étudiant) ; à ajouter si la grille de la section 0.0 demande deux diagrammes de ce type.

## 4.6 Structure Spring Boot et correspondance UML → code (critères 6 et 7)

Chaque service du dossier `services/` reprend le même découpage en packages ; c'est ce que la grille appelle « architecture en couches ».

```text
com.robotedu.api
├── model/        entités JPA (@Entity) : User, Course, Session, Attempt…
├── repository/   interfaces JpaRepository<Entity, Long>
├── service/      logique métier (@Service, @Transactional)
├── controller/   endpoints REST (@RestController)
├── dto/          objets échangés avec le client (records Java)
├── security/     filtre JWT, SecurityFilterChain
├── config/       beans, CORS, OpenAPI
├── exception/    exceptions métier + @RestControllerAdvice
└── event/        événements publiés sur Redis Streams
```

| Élément UML | Réalisation Java / Spring | Package |
| --- | --- | --- |
| Classe du diagramme 4.3 | classe `@Entity` + `@Table` | `model` |
| Association `1 — *` | `@OneToMany` / `@ManyToOne` | `model` |
| Attribut `enabled` | champ `boolean`, colonne `enabled` | `model` |
| Lifeline « Controller » | classe `@RestController` | `controller` |
| Lifeline « Service » | classe `@Service` | `service` |
| Lifeline « Repository » | interface `JpaRepository` | `repository` |
| Message `submit(dto, userId)` | méthode `AttemptService.submit` | `service` |
| Nœud de décision | `if` + exception métier | `service`, `exception` |
| Valeur énumérée | `enum` Java stocké en `@Enumerated(STRING)` | `model` |

## 4.7 Extraits de code : entité, dépôt, service (critères 6 et 8)

Entité `Attempt`, relation `@ManyToOne` vers `Session` et `Exercise`, et `@OneToOne` vers `Score` :

```java
@Entity
@Table(name = "attempt")
public class Attempt {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private Session session;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    private Exercise exercise;

    @OneToOne(mappedBy = "attempt", cascade = CascadeType.ALL)
    private Score score;

    @Enumerated(EnumType.STRING)
    private AttemptStatus status;

    private Instant submittedAt = Instant.now();
    // getters et setters
}
```

Dépôt, avec une requête dérivée du nom de la méthode :

```java
public interface AttemptRepository extends JpaRepository<Attempt, Long> {
    List<Attempt> findBySessionUserIdOrderBySubmittedAtDesc(Long userId);
}
```

Service, qui suit exactement le diagramme d'activité 4.5 :

```java
@Service
public class AttemptService {
    private final ExerciseRepository exercises;
    private final AttemptRepository attempts;
    private final GradingService grading;

    public AttemptService(ExerciseRepository exercises, AttemptRepository attempts, GradingService grading) {
        this.exercises = exercises;
        this.attempts = attempts;
        this.grading = grading;
    }

    @Transactional
    public AttemptDto submit(SubmitAttemptDto dto, Long userId) {
        Exercise exercise = exercises.findById(dto.exerciseId())
            .orElseThrow(() -> new ResourceNotFoundException("Exercise " + dto.exerciseId()));
        ScoreResult result = grading.grade(exercise, dto);
        Attempt attempt = new Attempt(userId, exercise, dto.sessionId());
        attempt.setScore(new Score(attempt, result));
        attempt.setStatus(result.value() >= exercise.getPassingScore()
            ? AttemptStatus.PASSED : AttemptStatus.FAILED);
        return AttemptDto.from(attempts.save(attempt));
    }
}
```

L'injection se fait par le constructeur (pas de `@Autowired` sur les champs) : le service se teste avec Mockito sans démarrer Spring.

## 4.8 API REST et gestion des erreurs (critères 8 et 9)

Les endpoints du MVP réduit. Tous sont documentés automatiquement par springdoc-openapi (page Swagger) et préfixés par `/api`.

| Méthode | Chemin | Rôle | Accès | Succès |
| --- | --- | --- | --- | --- |
| POST | `/auth/register` | créer un compte | public | 201 |
| POST | `/auth/login` | obtenir un JWT | public | 200 |
| GET | `/courses` | lister les cours publiés | STUDENT | 200 |
| GET | `/courses/{id}` | détail avec leçons | STUDENT | 200 |
| POST | `/courses` | créer un cours | ADMIN | 201 |
| PUT | `/courses/{id}` | modifier un cours | ADMIN | 200 |
| DELETE | `/courses/{id}` | supprimer un cours | ADMIN | 204 |
| POST | `/enrollments` | s'inscrire à un cours | STUDENT | 201 |
| POST | `/sessions` | démarrer une session (clé d'idempotence) | STUDENT | 201 |
| DELETE | `/sessions/{id}` | arrêter une session | STUDENT | 204 |
| POST | `/attempts` | soumettre une tentative | STUDENT | 201 |
| GET | `/attempts/me` | mes tentatives et scores | STUDENT | 200 |

Les quatre opérations CRUD complètes sont sur `Course` ; le reste est du métier.

| Erreur | Exception | Code HTTP |
| --- | --- | --- |
| DTO invalide | `MethodArgumentNotValidException` | 400 |
| Token absent ou expiré | `AuthenticationException` | 401 |
| Rôle insuffisant | `AccessDeniedException` | 403 |
| Ressource absente | `ResourceNotFoundException` | 404 |
| Doublon (e-mail, inscription) | `ConflictException` | 409 |

Une classe `GlobalExceptionHandler` annotée `@RestControllerAdvice` renvoie toujours le même JSON : `{ "timestamp", "status", "error", "message", "path" }`.

## 4.9 Plan de tests : JUnit, MockMvc, Postman (critères 8 et 9)

| Niveau | Outil | Ce qu'on vérifie | Responsable |
| --- | --- | --- | --- |
| Unitaire | JUnit 5 + Mockito | `AttemptService`, les quatre correcteurs, calcul du score | D |
| Web | MockMvc + `@WebMvcTest` | codes 400, 401, 403, 404 de chaque controller | D |
| Persistance | `@DataJpaTest` (H2) | requêtes des dépôts, contraintes d'unicité | D |
| Intégration | Testcontainers (Postgres) | parcours inscription → session → tentative | C |
| Manuel | collection Postman | un scénario par endpoint du tableau 4.8 | D |

La collection Postman suit l'ordre de la démo : `register`, `login` (le token est stocké dans une variable), `GET /courses`, `POST /enrollments`, `POST /sessions`, `POST /attempts`, `GET /attempts/me`. Chaque requête contient au moins un test `pm.test` sur le code de retour.

Objectif de couverture : 70 % des lignes sur `service`, mesuré par JaCoCo dans la CI.

## 4.10 Documentation technique, qualité du code, oral (critères 1, 2, 10)

**Documentation à rendre.** README (installation en trois commandes, variables d'environnement), page Swagger, ce dossier UML (diagrammes 0.2, 4.3, 4.4, 4.5), collection Postman exportée, script de base de données Flyway.

**Qualité du code.** Spotless et la CI refusent un push mal formaté ; pas de logique métier dans un controller ; pas d'entité renvoyée directement au client (toujours un DTO) ; noms en anglais, commentaires utiles seulement.

**Oral et démo.** Chaque membre présente la partie qu'il a codée (matrice du chapitre 22) et doit savoir expliquer une ligne de son code. Démo en cinq minutes sur le parcours : connexion → choix d'un cours → session 3D → soumission → score affiché. Le plan B est une vidéo de ce parcours enregistrée en semaine 12 (gel du 16/01).

**Points à confirmer avec l'enseignant** : base H2 ou Postgres acceptée, nombre exact de diagrammes d'activité attendus, Python autorisé dans l'environnement d'exercice.
