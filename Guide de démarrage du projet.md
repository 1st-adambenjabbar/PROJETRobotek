# Guide de démarrage du projet

Oct 15, 2026 · @adam

À lire en entier avant d'écrire la première ligne de code : ce guide explique la plateforme de robotique, les outils et les règles de l'équipe, sans rien supposer. Il fait une vingtaine de pages ; le document d'architecture complet reste la référence pour le détail technique.

## 1. Le projet en 5 minutes

Nous construisons un site web qui apprend la robotique par la pratique, comme The Construct : l'étudiant ouvre un cours, écrit du code, et voit un robot bouger dans un monde en 3D, directement dans son navigateur. Il n'y a rien à installer chez lui.

### 1.1 Ce que voit l'étudiant

1. Il se connecte avec son e-mail et un mot de passe.
2. Il choisit un cours (par exemple « Premiers pas avec ROS 2 ») et s'y inscrit.
3. Il ouvre une leçon : le texte à gauche, un éditeur de code et un terminal à droite, une scène 3D en dessous.
4. Il clique sur « Lancer ». Le site démarre pour lui seul un petit ordinateur virtuel (un conteneur) avec un robot simulé.
5. Il écrit un programme, par exemple « avance jusqu'au mur puis tourne », et regarde le robot l'exécuter dans la 3D.
6. Il clique sur « Soumettre » : le site vérifie le résultat, donne une note et un retour (« le robot s'est arrêté à 0,3 m du mur, bravo »).

### 1.2 Ce qu'on a le droit de faire et ce qu'on a choisi

| Choix | Pourquoi |
| --- | --- |
| Simulation 3D avec Three.js, sans Gazebo | Gazebo demande une carte graphique sur le serveur ; Three.js dessine dans le navigateur de l'étudiant |
| Un conteneur par étudiant | le code d'un étudiant ne peut pas casser celui d'un autre |
| Application en Java (Spring Boot) | le cours impose Java et Spring Boot pour la notation |
| Python uniquement dans les exercices | c'est le langage que l'étudiant utilise pour programmer le robot, pas celui du site |
| MVP réduit | peu de fonctions, mais qui marchent de bout en bout, plutôt que beaucoup de fonctions à moitié faites |

MVP veut dire « produit minimum viable » : la plus petite version du site qui permet vraiment de faire le parcours ci-dessus. Tout ce qui n'est pas dans ce parcours (classement des étudiants, forum, cartes SLAM avancées) est repoussé.

### 1.3 Ce que vous devez retenir

- Le site a **deux mondes séparés** : le site lui-même (comptes, cours, notes), écrit en Java, et le monde de l'exercice (robot, simulation), qui tourne dans un conteneur et utilise ROS 2 et Python.
- Le navigateur de l'étudiant fait le **dessin 3D** ; le serveur ne dessine jamais rien.
- Chacun de nous quatre possède des briques précises (section 10) : on ne touche pas au code des autres sans en parler.

## 2. Comment marche un site web

Tout le projet repose sur quelques idées simples. Si tu les as en tête, le reste du guide devient lisible.

### 2.1 Client et serveur

Un site web, ce sont deux programmes qui se parlent. Le **client** est le navigateur (Chrome, Firefox) : il affiche les pages et envoie des demandes. Le **serveur** est un programme qui tourne en continu sur une machine : il reçoit les demandes, travaille, et répond. Le client demande toujours en premier ; le serveur ne parle jamais sans qu'on lui demande.

&#91;embedded content: navigateur, serveur et base de données · un aller-retour\]

Le serveur n'a pas de mémoire entre deux demandes. Pour savoir qui est connecté, le navigateur renvoie un **jeton** (JWT) à chaque demande, comme un badge qu'on montre à chaque porte.

### 2.2 HTTP et les verbes

La langue commune du client et du serveur s'appelle **HTTP**. Une demande contient un verbe, une adresse et parfois des données.

| Verbe | Sens | Exemple dans le projet |
| --- | --- | --- |
| `GET` | lire | `GET /api/courses` donne la liste des cours |
| `POST` | créer ou déclencher | `POST /api/sessions` démarre une session |
| `PUT` | remplacer | `PUT /api/courses/3` modifie le cours 3 |
| `DELETE` | supprimer | `DELETE /api/sessions/12` arrête une session |

La réponse commence par un **code de statut** qui dit comment ça s'est passé :

| Code | Signification |
| --- | --- |
| 200 / 201 / 204 | ça a marché / créé / fait, rien à renvoyer |
| 400 | la demande est mal formée (champ manquant) |
| 401 | tu n'es pas connecté |
| 403 | tu es connecté mais tu n'as pas le droit |
| 404 | la chose demandée n'existe pas |
| 500 | le serveur a planté : c'est un bug chez nous |

### 2.3 API REST et JSON

Une **API** est la liste des demandes que le serveur accepte. Elle est dite **REST** quand chaque adresse désigne une chose (`/courses`, `/sessions`) et que le verbe dit ce qu'on en fait. Les données voyagent en **JSON**, un texte facile à lire pour un humain comme pour un programme :

```json
{
  "id": 3,
  "title": "Premiers pas avec ROS 2",
  "level": "beginner",
  "published": true
}
```

En Java, une telle structure s'appelle un **DTO** (objet de transfert). On ne renvoie jamais directement une ligne de la base : on la copie dans un DTO pour ne montrer que ce qu'on veut (jamais le mot de passe, par exemple).

### 2.4 Base de données

Une **base de données** est un classeur géant que le serveur consulte. Elle est organisée en **tables** (une par type de chose : `user`, `course`, `attempt`), chaque table a des **colonnes** (`email`, `title`) et des **lignes** (une par élément). Une ligne en pointe une autre grâce à une **clé étrangère** : `attempt.exercise_id` contient l'identifiant de l'exercice concerné. Nous utilisons PostgreSQL et un langage de requêtes, le SQL :

```sql
SELECT title FROM course WHERE published = true ORDER BY title;
```

En pratique, nous n'écrirons presque pas de SQL : Spring Data JPA le génère à partir de nos classes Java (section 6).

## 3. Notre architecture

Le schéma ci-dessous est le plan de toute la plateforme. Il se lit de haut en bas : ce que l'étudiant voit, puis la porte d'entrée, puis les services, puis ce sur quoi ils s'appuient.

&#91;embedded content: architecture globale · 3 services Java et 1 conteneur par session\]

### 3.1 Lecture du schéma

- **Navigateur** : l'étudiant ne voit que lui. Il affiche les pages et dessine la scène 3D. Il ne parle qu'à Caddy.
- **Caddy** : un petit serveur qui reçoit tout et redirige. C'est la seule porte ouverte vers l'extérieur, ce qui simplifie la sécurité.
- **api (Spring Boot)** : le cœur du site. Il gère les comptes, les cours, les inscriptions, les tentatives et les notes. C'est lui qui parle à la base de données.
- **session-manager** : il ne fait qu'une chose, créer et détruire les conteneurs. Quand l'étudiant clique sur « Lancer », l'api lui transmet l'ordre.
- **realtime-proxy** : il relie le navigateur au robot de la bonne session, et vérifie à chaque message que l'étudiant a le droit de parler à ce robot.
- **Conteneur de session** : un mini-ordinateur jetable, un par étudiant, qui contient ROS 2 et le simulateur. C'est le seul endroit où le code d'un étudiant s'exécute, donc le plus verrouillé.
- **PostgreSQL + Redis** : PostgreSQL garde les données durables ; Redis transmet des messages rapides entre services (par exemple « tentative soumise »).
- **Docker Engine** : le programme de la machine qui lance les conteneurs.

### 3.2 Que se passe-t-il quand l'étudiant clique sur « Lancer » ?

1. Le navigateur envoie `POST /api/sessions` avec son jeton.
2. L'api vérifie le jeton, vérifie qu'il n'a pas déjà une session, et écrit la session dans la base avec l'état `REQUESTED`.
3. L'api demande au session-manager de démarrer un conteneur ; celui-ci le crée avec Docker et passe l'état à `RUNNING`.
4. Le navigateur ouvre une connexion WebSocket vers le realtime-proxy, qui l'aiguille vers le conteneur.
5. Les positions du robot arrivent en continu au navigateur, qui les dessine avec Three.js. Les ordres de l'étudiant font le chemin inverse.

### 3.3 Pourquoi trois services et pas un seul ?

On sépare un morceau quand il a une raison de sécurité ou de charge différente : seul le session-manager a le droit de piloter Docker (donc de prendre la main sur la machine), et le proxy gère beaucoup de connexions longues. Tout le reste (comptes, cours, notes) reste ensemble dans l'api, ce qui est bien plus simple pour des débutants.

## 4. Les technologies, une par une

Beaucoup de noms vont revenir. Voici, pour chacun, ce que c'est, pourquoi on l'utilise et ce que tu as besoin d'en savoir au début. Tu n'as pas à tout maîtriser : tu ne vas toucher en profondeur que les outils de ta partie.

### 4.1 Côté site (notre code Java)

| Outil | C'est quoi | Pourquoi ici | À savoir d'abord |
| --- | --- | --- | --- |
| Java 21 | le langage de programmation du site | imposé par le cours | classes, méthodes, listes, `Optional`, `record` |
| Spring Boot | un cadre qui fournit un serveur web prêt à l'emploi | on écrit seulement notre logique ; Spring gère le reste | les annotations `@RestController`, `@Service` |
| Maven | l'outil qui télécharge les bibliothèques et compile | standard avec Spring | `mvn test`, `mvn spring-boot:run` |
| Spring Data JPA / Hibernate | traduit nos classes Java en lignes de base de données | évite d'écrire du SQL à la main | `@Entity`, `@ManyToOne` |
| Flyway | applique des scripts SQL de migration dans l'ordre | la base évolue sans casser celle des autres | un fichier `V2__...sql` par changement |
| Spring Security + JWT | vérifie qui est connecté et ce qu'il peut faire | protège les endpoints | le rôle `STUDENT` ou `ADMIN` |
| springdoc-openapi | génère une page Swagger qui documente l'API | sert de démo et de documentation | on ouvre `/swagger-ui.html` |
| JUnit 5, Mockito, MockMvc | outils de test | 10 points de la grille portent sur les tests | un test = une méthode `@Test` |
| Postman | outil pour envoyer des requêtes à la main | requis par le cours | une collection de requêtes enregistrées |

### 4.2 Données et messages

| Outil | Rôle | Remarque |
| --- | --- | --- |
| PostgreSQL | la base de données durable | H2 (base en mémoire) pour les tests, à confirmer avec l'enseignant pour le rendu |
| Redis | messages rapides entre services (« tentative soumise ») | on n'y garde rien d'important |

### 4.3 Côté navigateur

| Outil | Rôle |
| --- | --- |
| React + TypeScript | construit les pages et écrans |
| Three.js | dessine la scène 3D (le sol, les murs, le robot) |
| Monaco | l'éditeur de code intégré (celui de VS Code) |
| xterm.js | le terminal intégré |

### 4.4 Le monde de l'exercice (le robot)

| Outil | Rôle |
| --- | --- |
| ROS 2 Jazzy | la boîte à outils standard des robots : fait communiquer les programmes du robot |
| PyBullet | un moteur de physique : calcule les collisions, les roues, le LiDAR |
| rosbridge | traduit ROS 2 en WebSocket pour que le navigateur puisse l'écouter |
| URDF / xacro | fichiers XML qui décrivent le robot (corps, roues, capteurs) |
| Python | le langage dans lequel l'étudiant programme son robot |

### 4.5 Outils d'équipe

| Outil | Rôle |
| --- | --- |
| Git + GitLab | historique du code, branches, revues (section 7) |
| Docker + Compose | lance tout le projet avec une commande (section 8) |
| Caddy | porte d'entrée et HTTPS |
| IntelliJ IDEA | l'IDE conseillé pour Java et Spring Boot |
| VS Code | pratique pour le web, le XML et Python |

Une règle utile : si un mot t'est inconnu, cherche-le dans le glossaire (section 14) avant de demander. Si le glossaire ne le contient pas, c'est qu'il manque : dis-le, on l'ajoutera.

## 5. La simulation 3D expliquée simplement

C'est la partie la plus spectaculaire du projet, et celle qui fait le plus peur. Elle repose en réalité sur trois idées : un robot qui publie des mesures, un pont qui les transporte, et un navigateur qui les dessine.

### 5.1 ROS 2 en quatre mots

- **Node** : un petit programme qui fait une chose (lire le LiDAR, commander les roues).
- **Topic** : un « canal » nommé sur lequel les nodes échangent, comme un salon de discussion (`/scan`, `/cmd_vel`).
- **Message** : le contenu envoyé sur un topic, avec un type précis (une vitesse, une liste de distances).
- **Publier / s'abonner** : un node publie sur un topic ; tous les nodes abonnés le reçoivent. L'émetteur ne sait pas qui écoute.

| Topic | Sens | Contenu |
| --- | --- | --- |
| `/cmd_vel` | étudiant → robot | vitesse voulue : avancer, tourner |
| `/scan` | robot → étudiant | distances mesurées par le LiDAR |
| `/odom` | robot → étudiant | position estimée d'après les roues |
| `/tf` | robot → étudiant | position de chaque pièce du robot dans l'espace |
| `/map` | robot → étudiant | carte construite (exercices avancés) |

Un **LiDAR** est un capteur qui tourne et mesure la distance au premier obstacle dans de nombreuses directions. Le résultat est une liste de distances, un peu comme une chauve-souris qui « entend » la pièce. Dans la simulation, PyBullet lance des rayons virtuels pour fabriquer cette liste.

### 5.2 Pourquoi pas Gazebo

Gazebo est le simulateur classique de ROS, mais il dessine l'image sur le serveur et demande donc une carte graphique par étudiant. Notre simulateur calcule seulement la physique (qui est légère) et laisse le navigateur faire le dessin. Du côté de ROS, rien ne change : mêmes topics, mêmes messages.

### 5.3 Le trajet complet

&#91;embedded content: chaîne de la simulation · du robot à l'écran et retour\]

1. `sim_node` fait avancer la physique et publie `/scan`, `/odom`, `/tf`.
2. `rosbridge` écoute ces topics et envoie leur contenu en JSON sur une connexion WebSocket.
3. Le `realtime-proxy` (en Java) contrôle que le navigateur a le droit de voir cette session, puis transmet.
4. Le navigateur reçoit les messages et déplace le robot dans la scène 3D.
5. L'étudiant tape un programme Python dans le terminal ; il publie `/cmd_vel` ; le robot bouge ; la boucle recommence.

Un **WebSocket** est une connexion qui reste ouverte, contrairement à HTTP où chaque demande est suivie d'une réponse puis d'un silence. C'est ce qui permet de recevoir des positions en continu.

### 5.4 Three.js en quatre mots

- **Scene** : le monde 3D, vide au départ.
- **Mesh** : un objet visible (géométrie + matériau) : le sol, un mur, un robot.
- **Camera** : le point de vue.
- **Renderer** : le programme qui dessine la scène dans une zone de la page, de nombreuses fois par seconde.

```ts
const scene = new THREE.Scene();
const robot = new THREE.Mesh(
  new THREE.BoxGeometry(0.4, 0.2, 0.3),
  new THREE.MeshStandardMaterial({ color: 0x3377ff })
);
scene.add(robot);

// à chaque message /odom reçu du serveur :
function onOdom(msg) {
  robot.position.set(msg.x, 0, msg.y);
}
```

Tout le travail du côté navigateur consiste à recevoir des nombres et à mettre à jour des objets comme celui-ci.

### 5.5 Comment on décrit un robot

Un robot est décrit dans un fichier **URDF** : une liste de pièces (« links » : chaîssis, roues, capteur) reliées par des articulations (« joints »). **Xacro** est une version de l'URDF avec des variables et des répétitions pour ne pas copier-coller quatre fois la même roue. Ce sont des fichiers XML ; on les modifie dans l'IDE et on les teste en lançant le robot.

### 5.6 Ce que l'étudiant écrit, lui

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class Avance(Node):
    def __init__(self):
        super().__init__('avance')
        self.pub = self.create_publisher(Twist, '/cmd_vel', 10)
        self.create_timer(0.1, self.tick)

    def tick(self):
        msg = Twist()
        msg.linear.x = 0.2   # 0,2 m/s vers l'avant
        self.pub.publish(msg)

rclpy.init()
rclpy.spin(Avance())
```

C'est le seul endroit du projet où Python est utilisé par les étudiants. Le site, lui, reste en Java.

## 6. Java et Spring Boot pour débutants

C'est ici que se joue la plus grande partie de la note (40 points sur 100 portent sur le code Spring Boot). L'idée centrale est de **ranger le code en couches**, chacune avec un seul rôle.

### 6.1 Les quatre couches

| Couche | Annotation | Rôle | Ce qu'elle ne doit jamais faire |
| --- | --- | --- | --- |
| Controller | `@RestController` | reçoit la requête HTTP, renvoie la réponse | contenir de la logique métier |
| Service | `@Service` | applique les règles (« un étudiant ne peut pas avoir 2 sessions ») | parler HTTP |
| Repository | `JpaRepository` | lit et écrit dans la base | contenir des règles |
| Entity | `@Entity` | une classe = une table | porter du code de service |

Une requête traverse ces couches dans l'ordre : controller, service, repository, base de données, puis le chemin inverse pour la réponse. C'est exactement le diagramme de séquence que le cours demande.

### 6.2 Un exemple complet, couche par couche

**Entité** (une ligne de la table `course`) :

```java
@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private boolean published;
    // getters et setters
}
```

**Repository** (aucun SQL à écrire) :

```java
public interface CourseRepository extends JpaRepository<Course, Long> {
    List<Course> findByPublishedTrue();
}
```

**Service** (la logique) :

```java
@Service
public class CourseService {
    private final CourseRepository courses;

    public CourseService(CourseRepository courses) {
        this.courses = courses;
    }

    public List<CourseDto> listPublished() {
        return courses.findByPublishedTrue().stream()
                .map(CourseDto::from)
                .toList();
    }
}
```

**Controller** (l'adresse HTTP) :

```java
@RestController
@RequestMapping("/api/courses")
public class CourseController {
    private final CourseService service;

    public CourseController(CourseService service) {
        this.service = service;
    }

    @GetMapping
    public List<CourseDto> list() {
        return service.listPublished();
    }
}
```

Maintenant, `GET /api/courses` renvoie la liste en JSON. Quatre petites classes suffisent : c'est tout l'intérêt de Spring Boot.

### 6.3 Les mots qui reviennent

- **Annotation** : une étiquette `@Quelquechose` qui dit à Spring comment traiter une classe ou une méthode.
- **Injection de dépendances** : tu ne fais jamais `new CourseService()` toi-même ; Spring le crée et le passe au constructeur du controller. On le voit dans le constructeur ci-dessus.
- **DTO** : classe ou `record` qui décrit ce qu'on envoie au client (section 2.3).
- **Validation** : `@NotBlank`, `@Size`... sur un DTO ; Spring refuse la requête avec un 400 si elle ne respecte pas les règles.
- **Exception métier** : par exemple `ResourceNotFoundException`, qu'une classe `@RestControllerAdvice` transforme en réponse 404 propre.

### 6.4 Lancer et tester

```bash
mvn spring-boot:run    # démarre le serveur sur le port 8000
mvn test               # lance tous les tests
```

Une fois lancé, ouvre `/swagger-ui.html` : tu y vois tous les endpoints et tu peux les essayer depuis le navigateur. C'est aussi l'outil de la démo.

### 6.5 Un test minimal

```java
@ExtendWith(MockitoExtension.class)
class CourseServiceTest {
    @Mock CourseRepository courses;
    @InjectMocks CourseService service;

    @Test
    void listPublished_returnsOnlyPublishedCourses() {
        Course c = new Course();
        c.setTitle("ROS 2");
        when(courses.findByPublishedTrue()).thenReturn(List.of(c));

        assertEquals(1, service.listPublished().size());
    }
}
```

Le test remplace la vraie base par un faux dépôt (le **mock**), donc il tourne en une fraction de seconde. Ce genre de test est ce que la grille attend.

## 7. Git et GitLab : le workflow quotidien

**Git** garde l'historique du code et permet à quatre personnes de modifier le même projet sans s'écraser. **GitLab** est le site qui héberge le dépôt commun, affiche les demandes de fusion et lance les tests automatiquement.

### 7.1 Les mots

| Mot | Sens |
| --- | --- |
| dépôt (repository) | le dossier du projet avec tout son historique |
| commit | une photo du code avec un message qui dit ce qui a changé |
| branche | une version parallèle où on travaille sans gêner les autres |
| `main` | la branche stable ; on n'y pousse jamais directement |
| merge request (MR) | la demande « merci de relire et fusionner ma branche dans `main` » |
| conflit | deux personnes ont modifié la même ligne ; il faut choisir |

### 7.2 Le cycle à suivre pour chaque tâche

```bash
git checkout main && git pull                   # 1. repartir de la dernière version
git checkout -b feature/course-list             # 2. créer sa branche
# ... coder, tester ...
git add .
git commit -m "Add GET /api/courses endpoint"  # 3. commit clair, en anglais
git push -u origin feature/course-list          # 4. envoyer sur GitLab
```

Ensuite, sur GitLab : ouvrir une **merge request**, assigner un relecteur, attendre que les tests passent au vert, corriger ses remarques, puis fusionner. Après la fusion, retour à l'étape 1.

### 7.3 Règles simples

- Une branche = une tâche courte (une à trois journées). Plus longue, elle devient impossible à fusionner.
- Nom de branche : `feature/...` pour du neuf, `fix/...` pour une correction.
- Un commit = une idée. Pas de « modifs » ou « a » comme message.
- Chaque merge request est relue par un autre membre avant d'être fusionnée : c'est ainsi qu'on apprend le code des autres et qu'on évite les bugs.
- On ne met jamais de mot de passe ou de clé dans Git. Le fichier `.env` n'est pas versionné ; seul `.env.example` l'est.
- Tirer `main` régulièrement (`git pull`) évite les gros conflits.

### 7.4 Quand il y a un conflit

Git écrit dans le fichier des marqueurs :

```text
<<<<<<< HEAD
version de main
=======
ta version
>>>>>>> feature/course-list
```

Ouvre le fichier, garde le bon morceau (ou combine les deux), supprime les marqueurs, puis `git add` et `git commit`. Si tu hésites, appelle la personne qui a modifié l'autre version : deux minutes de discussion valent mieux qu'une mauvaise fusion.

## 8. Docker : les commandes dont on a besoin

Un **conteneur** est une boîte isolée qui contient un programme et tout ce dont il a besoin pour tourner. Une **image** est le modèle à partir duquel on crée des conteneurs, comme un plan de maison pour construire plusieurs maisons identiques. **Docker Compose** lance plusieurs conteneurs ensemble à partir d'un seul fichier (`compose.yaml`).

Pourquoi c'est utile pour nous : tout le monde a exactement le même environnement (même version de PostgreSQL, de Redis, de ROS 2), et chaque étudiant de la plateforme reçoit son propre conteneur.

### 8.1 Commandes à connaître

| Commande | Effet |
| --- | --- |
| `docker compose up -d` | démarre tous les services en arrière-plan |
| `docker compose ps` | liste ce qui tourne |
| `docker compose logs -f api` | affiche en direct les messages du service `api` |
| `docker compose restart api` | redémarre un service |
| `docker compose down` | arrête et supprime les conteneurs (les données de la base restent) |
| `docker compose build` | reconstruit les images après une modification |
| `docker exec -it <nom> bash` | ouvre un terminal dans un conteneur |
| `docker system df` | montre l'espace disque utilisé |

### 8.2 Services démarrés en développement

`db` (PostgreSQL), `redis`, `api`, `session-manager`, `realtime-proxy`, `web`. L'image du robot est construite par Compose mais lancée uniquement par le session-manager.

### 8.3 Problèmes classiques

- **« port already in use »** : un autre programme occupe le port. Arrête-le ou change le port dans `.env`.
- **« le service démarre puis s'arrête »** : lis `docker compose logs <service>`, la raison y est presque toujours.
- **« ça marche chez toi mais pas chez moi »** : reconstruis avec `docker compose build --no-cache`, puis relance.
- **Disque plein** : `docker system prune` supprime les images et conteneurs inutilisés (demande avant, ce n'est pas annulable).

### 8.4 Une précaution de sécurité

Le session-manager a besoin du socket Docker pour créer des conteneurs, ce qui lui donne beaucoup de pouvoir sur la machine. En développement c'est acceptable ; pour la démo, on le restreint (voir le document d'architecture, chapitre 16). Aucun autre service ne monte ce socket.

## 9. Installer son poste : checklist pas à pas

À faire dans l'ordre, et à cocher. Si une étape échoue, copie le message d'erreur exact dans le groupe : à plusieurs, on résout en dix minutes ce qui prend une heure tout seul.

### 9.1 Logiciels

- [ ] **Git** installé (`git --version`)
- [ ] **JDK 21** installé (`java -version` affiche 21)
- [ ] **Maven** installé, ou celui fourni par IntelliJ (`mvn -v`)
- [ ] **Docker** : Docker Desktop sous Windows ou macOS (avec WSL 2 sous Windows), Docker Engine sous Linux (`docker --version` et `docker compose version`)
- [ ] **Node.js 22** pour la partie web (`node -v`)
- [ ] **IntelliJ IDEA** (Java, Spring Boot) et **VS Code** (web, XML, Python)
- [ ] **Postman**

### 9.2 Compte GitLab et accès au dépôt

- [ ] Compte GitLab créé et invitation au projet acceptée
- [ ] Clé SSH créée et ajoutée dans GitLab (Préférences, puis Clés SSH)

```bash
ssh-keygen -t ed25519 -C "ton-email@exemple.fr"   # Entrée à chaque question
cat ~/.ssh/id_ed25519.pub                        # à copier dans GitLab
```

- [ ] Dépôt cloné : `git clone <adresse SSH du projet>`
- [ ] Nom et e-mail Git configurés :

```bash
git config --global user.name "Prénom Nom"
git config --global user.email "ton-email@exemple.fr"
```

### 9.3 Premier lancement

```bash
cp .env.example .env              # ta copie de la configuration
docker compose up -d db redis     # démarre la base et Redis
cd services/api
mvn spring-boot:run               # démarre l'api
```

Ouvre ensuite `http://localhost:8000/swagger-ui.html` : si la page s'affiche, ton poste est prêt. Le dépôt n'existe pas encore tant que l'équipe n'a pas créé le projet de départ (section 12) ; en attendant, fais les vérifications de la section 9.1.

### 9.4 Choix de l'IDE

IntelliJ IDEA est le plus confortable pour Java et Spring Boot : il propose l'auto-complétion, lance les tests d'un clic et débogue pas à pas. VS Code convient mieux au web et aux fichiers XML. Chacun choisit, mais on utilise tous les mêmes commandes dans le terminal (celles de ce guide).

## 10. Qui fait quoi et le calendrier des rendus

### 10.1 Les quatre responsabilités

Chacun possède des briques précises : il en est responsable du code, des tests et de la documentation. Cela ne veut pas dire travailler seul : chaque merge request est relue par un autre.

| Membre | Briques principales | En clair |
| --- | --- | --- |
| A | realtime-proxy | le lien en direct entre navigateur et robot |
| B | espace de travail (workspace) | la sauvegarde et la restauration des fichiers de l'étudiant |
| C | authentification, utilisateurs, sessions, sécurité, session-manager | les comptes, les jetons et le démarrage des conteneurs |
| D | catalogue, contenu, progression, notation, tests et Postman | les cours, les exercices, les scores et la qualité |

La répartition détaillée, y compris le front-end et la scène 3D, est dans le chapitre 22 du document d'architecture. Remplacez A, B, C, D par vos prénoms une fois la répartition confirmée ensemble.

### 10.2 Calendrier

La semaine 1 du projet est la semaine du 12 octobre 2026. Les trois rendus du cours sont fixes :

| Date | Étape |
| --- | --- |
| 1er novembre 2026 | premier rendu |
| 30 novembre 2026 | deuxième rendu |
| 16 janvier 2027 | gel du code : on ne code plus de nouvelle fonction |
| 20 janvier 2027 | rendu final |

Les semaines 52 et 53 sont des vacances : aucun travail prévu. Le contenu précis de chaque rendu est dans le tableau 0.0 du document d'architecture.

### 10.3 Comment on avance chaque semaine

1. Lundi : 15 minutes ensemble pour dire ce qu'on a fini, ce qu'on va faire et ce qui bloque.
2. En semaine : branches courtes, merge requests relues dans les 24 heures.
3. Vendredi : tout ce qui est terminé est fusionné dans `main`, et `main` doit toujours démarrer.

## 11. Ce que le cours exige

Le projet est noté sur 100 points avec la grille « Projet UML + Spring Boot » du professeur. Environ 40 points portent directement sur le code Spring Boot ; le reste note les diagrammes UML, les tests, la documentation et l'oral. Les points exacts de chaque critère sont dans la grille du professeur (`grilleEvaluation.pdf`) : relis-la.

### 11.1 Ce qu'il faut produire

| Livrable | Ce que c'est | Où c'est expliqué dans le document d'architecture |
| --- | --- | --- |
| Diagramme de cas d'utilisation | qui fait quoi avec le système (étudiant, admin) | section 0.2 |
| Diagramme de classes | les 10 entités et leurs relations | section 4.3 |
| Diagramme de séquence | l'échange entre controller, service, dépôt et base | sections 0.2 et 4.4 |
| Diagramme d'activité | l'algorithme d'une fonction, avec décisions | section 4.5 |
| Application Spring Boot en couches | `model`, `repository`, `service`, `controller`, `dto`... | sections 4.6 et 4.7 |
| API REST documentée | endpoints, erreurs, Swagger | section 4.8 |
| Tests | JUnit, MockMvc, collection Postman | section 4.9 |
| Documentation et oral | README, dossier UML, démo | section 4.10 |

### 11.2 Les diagrammes UML en une phrase chacun

- **Cas d'utilisation** : des acteurs (les personnes) et ce qu'ils peuvent faire (se connecter, lancer une session).
- **Classes** : les « choses » du système, leurs champs, et comment elles se relient (un cours *possède* des leçons).
- **Séquence** : une conversation dans le temps entre des acteurs, de haut en bas.
- **Activité** : un organigramme avec des décisions (« si l'exercice n'existe pas, alors erreur 404 »).

Le diagramme de classes est le pont entre l'UML et le code : chaque classe du diagramme devient une classe `@Entity`, chaque trait devient une annotation (`@OneToMany`, `@ManyToOne`). Si le diagramme et le code se contredisent, c'est une perte de points.

### 11.3 Ce qui est réduit grâce au MVP

On ne code pas tout ce que le grand document décrit. Le périmètre minimal : connexion, catalogue de cours, inscription, session avec le robot, soumission d'un exercice et score. Le reste est des bonus, à ne faire que si le minimum est terminé et testé.

### 11.4 Points à confirmer avec l'enseignant

- Base H2 ou PostgreSQL acceptée pour le rendu.
- Nombre exact de diagrammes d'activité attendus.
- Python autorisé dans l'environnement d'exercice (le robot et ce que les étudiants écrivent), alors que l'application est en Java.

## 12. Semaine 0 : à faire avant d'écrire le vrai code

Ces exercices servent à vérifier que chacun a compris les bases. Chacun prend une heure ou deux. Ne passez pas à la suite tant qu'ils ne marchent pas pour tout le monde.

### 12.1 Exercice 1 : un serveur Java qui dit bonjour

Crée un projet sur start.spring.io (Maven, Java 21, dépendance Spring Web). Ajoute ce controller :

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Bonjour";
    }
}
```

Lance `mvn spring-boot:run`, ouvre `http://localhost:8080/hello`. Tu dois voir « Bonjour ». Tu viens de créer une API.

### 12.2 Exercice 2 : une table et un repository

Ajoute les dépendances Spring Data JPA et H2. Crée une entité `Course` (id, title), un `CourseRepository`, et un endpoint `POST /courses` qui enregistre un cours, `GET /courses` qui les liste. Teste avec Postman.

### 12.3 Exercice 3 : un premier test

Écris un test Mockito pour le service de l'exercice 2 (voir section 6.5). `mvn test` doit être vert.

### 12.4 Exercice 4 : Git en équipe

Crée une branche, un commit, une merge request ; fais relire par un collègue ; provoque volontairement un conflit sur une ligne et résous-le ensemble.

### 12.5 Exercice 5 : Docker

Lance PostgreSQL avec `docker run -e POSTGRES_PASSWORD=test -p 5432:5432 postgres:16`, connecte l'application de l'exercice 2 à cette base, puis arrête le conteneur et observe l'erreur.

### 12.6 Exercice 6 : un robot qui bouge

Avec l'image ROS 2 d'exercice (quand elle sera prête), lance le simulateur, publie une vitesse sur `/cmd_vel` depuis le terminal et regarde le robot avancer dans la 3D. Les détails sont dans le chapitre 8 du document d'architecture.

### 12.7 Critères de réussite de la semaine 0

- [ ] Chacun a lancé son propre serveur Spring Boot.
- [ ] Chacun a fait une merge request relue.
- [ ] Chacun sait ouvrir les logs d'un conteneur.
- [ ] On a une liste commune des erreurs rencontrées et de leur solution.

## 13. Règles d'équipe et erreurs fréquentes

### 13.1 Dix règles

1. On ne pousse jamais directement sur `main`.
2. Chaque tâche a sa branche et sa merge request.
3. Toute merge request est relue par quelqu'un d'autre avant la fusion.
4. `main` doit toujours démarrer et passer les tests.
5. Pas de mot de passe, de clé ou de jeton dans Git.
6. Un bug ou un blocage de plus d'une heure se dit dans le groupe.
7. On ne modifie pas la brique d'un autre sans le prévenir.
8. Les interfaces communes (les endpoints, les topics ROS, les événements) sont figées dans le document `contracts.md` ; on les change à plusieurs.
9. Une fonction n'est « finie » que si elle a un test et une ligne dans Swagger.
10. Le code du MVP passe avant les bonus.

### 13.2 Erreurs fréquentes des débutants

| Erreur | Pourquoi c'est un problème | Ce qu'il faut faire |
| --- | --- | --- |
| Mettre la logique dans le controller | impossible à tester, et la grille note les couches | déplacer dans le service |
| Renvoyer l'entité directement en JSON | expose le mot de passe, et casse quand la base change | renvoyer un DTO |
| Oublier `@Transactional` sur un service qui écrit | données à moitié enregistrées en cas d'erreur | l'ajouter sur la méthode |
| Modifier la base à la main | les autres n'ont pas le changement | écrire une migration Flyway |
| Gros commit de 40 fichiers | ne peut pas être relu | commits courts |
| Tester uniquement à la main | la régression revient sans bruit | écrire un test automatique |
| Copier-coller du code d'une IA sans le lire | on ne sait plus l'expliquer à l'oral | relire, comprendre, puis garder |
| Attendre le dernier moment pour les diagrammes UML | ils contredisent le code | les mettre à jour en même temps que le code |

### 13.3 Utiliser une IA correctement

Un assistant IA est utile pour expliquer une erreur, proposer un exemple ou relire un test. Mais l'oral du projet demande que chacun explique son propre code. Règle simple : si tu ne peux pas expliquer une ligne à voix haute, tu ne la gardes pas.

### 13.4 Comment demander de l'aide

Une bonne question contient : ce que tu voulais faire, la commande lancée, le message d'erreur complet, et ce que tu as déjà essayé. « Ça ne marche pas » ne permet pas d'aider.

## 14. Glossaire et FAQ

### 14.1 Glossaire

| Mot | Définition courte |
| --- | --- |
| API | liste des demandes qu'un serveur accepte |
| REST | style d'API où une adresse désigne une chose et le verbe HTTP l'action |
| JSON | format texte pour échanger des données |
| DTO | objet qui décrit ce qu'on envoie ou reçoit par l'API |
| Entité | classe Java reliée à une table de la base |
| JPA / Hibernate | outil qui convertit entités et lignes de base |
| Controller | classe qui reçoit les requêtes HTTP |
| Service | classe qui contient la logique métier |
| Repository | interface d'accès à la base |
| Injection de dépendances | Spring crée les objets et les passe aux constructeurs |
| JWT | jeton qui prouve qu'on est connecté |
| Migration | script qui fait évoluer le schéma de la base |
| Swagger | page qui documente et teste l'API |
| Mock | faux objet utilisé dans un test |
| Conteneur | boîte isolée qui exécute un programme |
| Image | modèle qui sert à créer des conteneurs |
| Docker Compose | lance plusieurs conteneurs à partir d'un fichier |
| Session | un robot simulé lancé pour un étudiant, dans son conteneur |
| MVP | version minimale qui marche de bout en bout |
| Merge request | demande de fusion d'une branche, avec relecture |
| ROS 2 | boîte à outils standard des robots |
| Node | programme ROS 2 qui fait une chose |
| Topic | canal nommé sur lequel les nodes échangent |
| LiDAR | capteur qui mesure des distances en tournant |
| URDF / xacro | fichiers XML décrivant un robot |
| PyBullet | moteur de physique de la simulation |
| rosbridge | pont qui expose ROS 2 en WebSocket |
| WebSocket | connexion qui reste ouverte pour échanger en continu |
| Three.js | bibliothèque de dessin 3D pour le navigateur |
| SLAM | méthode qui construit une carte tout en se localisant dedans |

### 14.2 Questions fréquentes

**Pourquoi Java si le robot est en Python ?** Parce que le cours impose Java et Spring Boot pour l'application. Le Python n'existe que dans l'environnement d'exercice, là où l'étudiant programme son robot.

**Dois-je tout comprendre de ROS 2 pour commencer ?** Non. Les quatre mots de la section 5.1 suffisent pour débuter. Les équipiers qui touchent à la simulation approfondiront ensuite.

**Pourquoi un conteneur par étudiant ?** Pour que le code d'un étudiant ne puisse pas perturber les autres ni l'hôte, et pour pouvoir le détruire proprement.

**Pourquoi pas Gazebo ?** Il demande une carte graphique par session. Three.js déplace le dessin chez l'étudiant.

**Que faire si je suis bloqué ?** Applique la règle de l'heure : cherche seul une heure, puis écris dans le groupe avec la question bien posée (13.4).

**Que lire après ce guide ?** Le document d'architecture complet : chapitre 2 pour la vue d'ensemble, chapitre 22 pour les responsabilités, et partie 4 pour tout ce que le cours demande en UML.
