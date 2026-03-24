# Formulaires et données utilisateur

# Étape 8 — Formulaires et données utilisateur

## 1. La théorie complète

### Comment fonctionne un formulaire HTML/PHP

```
Navigateur                    Serveur PHP
    │                              │
    │  1. Affiche le formulaire    │
    │ ◄────────────────────────── │
    │                              │
    │  2. Utilisateur remplit      │
    │     et soumet                │
    │                              │
    │  3. Envoie les données ─────►│
    │     via GET ou POST          │
    │                              │
    │  4. PHP traite et répond ───►│
    │ ◄────────────────────────── │
```

### **GET vs POST — la différence fondamentale**

```
GET  → Les données sont dans l'URL
       exemple.com/recherche.php?mot=php&page=2
       ↑ visibles, bookmarkables, limitées en taille

POST → Les données sont dans le corps de la requête
       Invisibles dans l'URL
       Pas de limite de taille pratique
       Pour les données sensibles (mot de passe, etc.)
```

```php
// Quand utiliser GET
// → Recherche, filtres, pagination — données non sensibles
// → L'URL peut être partagée/bookmarkée
exemple.com/articles.php?categorie=php&page=2

// Quand utiliser POST
// → Formulaire de connexion
// → Création/modification de données
// → Upload de fichiers
// → Données sensibles (mots de passe)
```

### **Les superglobales de formulaire**

```php
$_GET     // Données reçues via l'URL
$_POST    // Données reçues via le corps de la requête
$_REQUEST // Fusion de $_GET + $_POST (déconseillé — ambigu)
```

### **Récupérer les données**

```php
// Un formulaire POST avec un champ "email"
// <input type="text" name="email">

// Récupération basique — dangereux sans validation
$email = $_POST['email'];

// Récupération sécurisée avec valeur par défaut
$email = $_POST['email'] ?? '';

// Récupération depuis l'URL
// URL : page.php?id=42&tri=nom
$id  = $_GET['id']  ?? null;
$tri = $_GET['tri'] ?? 'date';
```

### **Vérifier si le formulaire a été soumis**

```php
// Méthode 1 — vérifier si $_POST n'est pas vide
if (!empty($_POST)) {
    // Le formulaire a été soumis
}

// Méthode 2 — vérifier la méthode HTTP (plus précis)
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Le formulaire a été soumis en POST
}

// Méthode 3 — vérifier un champ spécifique
if (isset($_POST['email'])) {
    // Le champ email existe dans la requête
}
```

### Validation des données

La validation c'est vérifier que les données **ont le bon format** :

```php
$erreurs = [];

// Vérifier qu'un champ n'est pas vide
if (empty(trim($_POST['nom'] ?? ''))) {
    $erreurs[] = "Le nom est obligatoire";
}

// Vérifier un email
if (!filter_var($_POST['email'] ?? '', FILTER_VALIDATE_EMAIL)) {
    $erreurs[] = "L'email est invalide";
}

// Vérifier un entier dans une plage
$age = (int)($_POST['age'] ?? 0);
if ($age < 18 || $age > 120) {
    $erreurs[] = "L'âge doit être entre 18 et 120";
}

// Vérifier une longueur minimale
$motdepasse = $_POST['motdepasse'] ?? '';
if (strlen($motdepasse) < 8) {
    $erreurs[] = "Le mot de passe doit faire au moins 8 caractères";
}

// Afficher les erreurs ou traiter
if (!empty($erreurs)) {
    // Il y a des erreurs — on les affiche
    foreach ($erreurs as $erreur) {
        echo "<p style='color:red'>{$erreur}</p>";
    }
} else {
    // Tout est valide — on traite
    echo "Formulaire valide !";
}
```

### Nettoyage des données — sécurité

Le nettoyage c'est **neutraliser les données dangereuses** avant de les utiliser :

```php
// ❌ Ne JAMAIS afficher directement ce que l'utilisateur envoie
echo $_POST['commentaire'];   // Faille XSS !

// ✅ htmlspecialchars — neutralise le HTML avant affichage
$commentaire = htmlspecialchars($_POST['commentaire'] ?? '', ENT_QUOTES, 'UTF-8');
echo $commentaire;

// ✅ trim — supprimer les espaces parasites
$nom = trim($_POST['nom'] ?? '');

// ✅ strip_tags — supprimer les balises HTML
$bio = strip_tags($_POST['bio'] ?? '');

// ✅ Caster le type pour les nombres
$age  = (int)($_POST['age'] ?? 0);
$prix = (float)($_POST['prix'] ?? 0.0);

// ✅ filter_var — filtre et valide en même temps
$email = filter_var(
    trim($_POST['email'] ?? ''),
    FILTER_SANITIZE_EMAIL    // Nettoie l'email
);

if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    $erreurs[] = "Email invalide";
}
```

---

### Le pattern PRG — Post Redirect Get

C'est **le pattern professionnel** pour gérer les formulaires. Il évite la double soumission quand l'utilisateur recharge la page.
```
Sans PRG :
  1. Utilisateur soumet le formulaire (POST)
  2. PHP traite et affiche "Merci"
  3. Utilisateur recharge la page → formulaire soumis UNE DEUXIÈME FOIS !

Avec PRG :
  1. Utilisateur soumet le formulaire (POST)
  2. PHP traite les données
  3. PHP redirige vers une autre page (GET)
  4. Utilisateur recharge → recharge juste la page de confirmation ✅
```

```php
<?php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // 1. Valider et traiter les données
    // 2. Sauvegarder en base de données
    // 3. Rediriger — JAMAIS afficher directement après un POST
    header('Location: confirmation.php');
    exit;   // ← INDISPENSABLE après header()
}
```

### Repopuler le formulaire après erreur

Quand il y a des erreurs, on réaffiche le formulaire avec les valeurs déjà saisies :

```php
// On récupère et sécurise la valeur pour la réafficher
$nom   = htmlspecialchars(trim($_POST['nom']   ?? ''), ENT_QUOTES, 'UTF-8');
$email = htmlspecialchars(trim($_POST['email'] ?? ''), ENT_QUOTES, 'UTF-8');
?>

<input type="text" name="nom" value="<?= $nom ?>">
<input type="email" name="email" value="<?= $email ?>">
```

## 2. Exemple concret et utile

Un **formulaire de contact** complet avec validation :

```php
<?php
declare(strict_types=1);

/**
 * contact.php
 * Formulaire de contact avec validation complète
 */

$erreurs  = [];
$succes   = false;

// Valeurs à repopuler si erreur
$nom     = '';
$email   = '';
$message = '';

// Traitement uniquement si formulaire soumis
if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    // Récupération et nettoyage
    $nom     = trim($_POST['nom']     ?? '');
    $email   = trim($_POST['email']   ?? '');
    $message = trim($_POST['message'] ?? '');

    // Validation du nom
    if (empty($nom)) {
        $erreurs['nom'] = "Le nom est obligatoire";
    } elseif (strlen($nom) < 2) {
        $erreurs['nom'] = "Le nom doit faire au moins 2 caractères";
    }

    // Validation de l'email
    if (empty($email)) {
        $erreurs['email'] = "L'email est obligatoire";
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $erreurs['email'] = "L'email n'est pas valide";
    }

    // Validation du message
    if (empty($message)) {
        $erreurs['message'] = "Le message est obligatoire";
    } elseif (strlen($message) < 10) {
        $erreurs['message'] = "Le message doit faire au moins 10 caractères";
    }

    // Si pas d'erreurs — traitement
    if (empty($erreurs)) {
        // En réalité : envoyer un email, sauvegarder en DB...
        $succes = true;

        // Pattern PRG — rediriger après succès
        header('Location: contact.php?succes=1');
        exit;
    }
}

// Vérifier si on arrive depuis une redirection de succès
$afficherSucces = isset($_GET['succes']) && $_GET['succes'] === '1';
?>
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Contact</title>
</head>
<body>

<?php if ($afficherSucces) : ?>
    <p style="color:green">Message envoyé avec succès !</p>
<?php endif; ?>

<?php if (!empty($erreurs)) : ?>
    <ul style="color:red">
        <?php foreach ($erreurs as $erreur) : ?>
            <li><?= htmlspecialchars($erreur, ENT_QUOTES, 'UTF-8') ?></li>
        <?php endforeach; ?>
    </ul>
<?php endif; ?>

<form method="POST" action="contact.php">

    <div>
        <label for="nom">Nom *</label>
        <input
            type="text"
            id="nom"
            name="nom"
            value="<?= htmlspecialchars($nom, ENT_QUOTES, 'UTF-8') ?>"
        >
    </div>

    <div>
        <label for="email">Email *</label>
        <input
            type="email"
            id="email"
            name="email"
            value="<?= htmlspecialchars($email, ENT_QUOTES, 'UTF-8') ?>"
        >
    </div>

    <div>
        <label for="message">Message *</label>
        <textarea
            id="message"
            name="message"
        ><?= htmlspecialchars($message, ENT_QUOTES, 'UTF-8') ?></textarea>
    </div>

    <button type="submit">Envoyer</button>

</form>

</body>
</html>
```

## 3. Bonnes pratiques et erreurs à éviter

```php
// ❌ Faire confiance aux données utilisateur
$id = $_GET['id'];
// Un utilisateur malveillant peut envoyer : ?id=<script>...

// ✅ Toujours valider et caster
$id = (int)($_GET['id'] ?? 0);
if ($id <= 0) {
    // Valeur invalide — on arrête
    http_response_code(400);
    exit('Requête invalide');
}

// ❌ Afficher les erreurs PHP en production
// Elles révèlent la structure de ton code aux attaquants
ini_set('display_errors', '1');

// ✅ En développement seulement
// En production : error_reporting(0) et logger dans un fichier

// ❌ Utiliser $_REQUEST
$email = $_REQUEST['email']; // Ambigu — vient de GET ou POST ?

// ✅ Être explicite
$email = $_POST['email'] ?? '';
```

## 4. Ce que font les seniors 👨‍💻

Les seniors **centralisent la validation** dans des fonctions dédiées :

```php
<?php
declare(strict_types=1);

/**
 * Valide et nettoie les données d'un formulaire
 *
 * @param array $donnees   Données brutes ($_POST)
 * @param array $regles    Règles de validation
 * @return array           ['erreurs' => [], 'donnees' => []]
 */
function validerFormulaire(array $donnees, array $regles): array
{
    $erreurs        = [];
    $donneesValides = [];

    foreach ($regles as $champ => $regle) {
        $valeur = trim($donnees[$champ] ?? '');

        // Obligatoire
        if (($regle['required'] ?? false) && empty($valeur)) {
            $erreurs[$champ] = $regle['message_required'] ?? "{$champ} est obligatoire";
            continue;
        }

        // Longueur minimale
        if (isset($regle['min_length']) && strlen($valeur) < $regle['min_length']) {
            $erreurs[$champ] = $regle['message_min'] ?? "{$champ} trop court";
            continue;
        }

        // Email
        if (($regle['email'] ?? false) && !filter_var($valeur, FILTER_VALIDATE_EMAIL)) {
            $erreurs[$champ] = $regle['message_email'] ?? "Email invalide";
            continue;
        }

        $donneesValides[$champ] = htmlspecialchars($valeur, ENT_QUOTES, 'UTF-8');
    }

    return ['erreurs' => $erreurs, 'donnees' => $donneesValides];
}

// Utilisation
if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    $regles = [
        'nom' => [
            'required'        => true,
            'min_length'      => 2,
            'message_required'=> "Le nom est obligatoire",
            'message_min'     => "Le nom doit faire au moins 2 caractères"
        ],
        'email' => [
            'required'        => true,
            'email'           => true,
            'message_required'=> "L'email est obligatoire",
            'message_email'   => "L'email n'est pas valide"
        ],
    ];

    $resultat = validerFormulaire($_POST, $regles);

    if (empty($resultat['erreurs'])) {
        // Traitement avec $resultat['donnees']
        header('Location: confirmation.php');
        exit;
    }

    $erreurs = $resultat['erreurs'];
}
```

## 🎯 Exercice 8

Tu dois créer un **formulaire d'inscription** pour une agence web.

Crée un fichier `inscription.php` qui contient le formulaire ET le traitement dans le même fichier :

### Le formulaire doit avoir ces champs :

- `prenom` — texte
- `nom` — texte
- `email` — email
- `mot_de_passe` — password
- `age` — nombre
- `departement` — select avec : `"Développement"`, `"Design"`, `"Marketing"`, `"RH"`

### La validation doit vérifier :

- `prenom` et `nom` — obligatoires, min 2 caractères
- `email` — obligatoire, format valide avec `filter_var`
- `mot_de_passe` — obligatoire, min 8 caractères
- `age` — obligatoire, entre 18 et 65
- `departement` — doit faire partie des valeurs autorisées

### Comportements attendus :

1. Si le formulaire n'est pas soumis → afficher le formulaire vide
2. Si erreurs → réafficher le formulaire avec les valeurs saisies et les erreurs en rouge
3. Si valide → afficher un message de succès avec les données nettoyées (jamais le mot de passe)
4. `htmlspecialchars` sur tout ce qui est affiché
5. Le champ `mot_de_passe` ne doit **jamais** être repopulé

Correction :

```php
<?php
declare(strict_types=1);

/**
 * inscription.php
 * Formulaire d'inscription avec validation complète
 */

// ============================================================
// CONFIGURATION
// ============================================================

const DEPARTEMENTS_AUTORISES = ['developpement', 'design', 'marketing', 'rh'];

// ============================================================
// INITIALISATION
// ============================================================

$erreurs = [];
$succes  = false;

// Valeurs à repopuler — toutes initialisées à vide
$prenom      = '';
$nom         = '';
$email       = '';
$age         = '';   // string vide — pas 0
$departement = '';

// Données nettoyées à afficher en cas de succès
$dataValideNettoye = [];

// ============================================================
// TRAITEMENT DU FORMULAIRE
// ============================================================

if ($_SERVER['REQUEST_METHOD'] === 'POST') {

    // --- Récupération et nettoyage brut ---
    $prenom      = trim($_POST['prenom']      ?? '');
    $nom         = trim($_POST['nom']         ?? '');
    $email       = trim($_POST['email']       ?? '');
    $password    = $_POST['password']         ?? '';   // pas de trim sur les mots de passe
    $ageRaw      = trim($_POST['age']         ?? '');  // brut pour tester si vide
    $departement = trim($_POST['departement'] ?? '');  // string simple, pas de []

    // --- Validation du prénom ---
    if (empty($prenom)) {
        $erreurs['prenom'] = "Le prénom est obligatoire";
    } elseif (strlen($prenom) < 2) {
        $erreurs['prenom'] = "Le prénom doit faire au moins 2 caractères";
    }

    // --- Validation du nom ---
    if (empty($nom)) {
        $erreurs['nom'] = "Le nom est obligatoire";
    } elseif (strlen($nom) < 2) {
        $erreurs['nom'] = "Le nom doit faire au moins 2 caractères";
    }

    // --- Validation de l'email ---
    if (empty($email)) {
        $erreurs['email'] = "L'email est obligatoire";
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $erreurs['email'] = "Le format de l'email n'est pas correct";
    }

    // --- Validation du mot de passe ---
    if (empty($password)) {
        $erreurs['password'] = "Le mot de passe est obligatoire";
    } elseif (strlen($password) < 8) {
        $erreurs['password'] = "Le mot de passe doit faire au moins 8 caractères";
    }

    // --- Validation de l'âge ---
    // On vérifie d'abord si le champ brut est vide
    // PUIS on caste en int pour vérifier la plage
    if ($ageRaw === '') {
        $erreurs['age'] = "L'âge est obligatoire";
    } else {
        $age = (int)$ageRaw;
        if ($age < 18 || $age > 65) {
            $erreurs['age'] = "L'âge doit être compris entre 18 et 65 ans";
        }
    }

    // --- Validation du département ---
    // Liste blanche — on vérifie que la valeur est autorisée
    if (empty($departement)) {
        $erreurs['departement'] = "Le département est obligatoire";
    } elseif (!in_array($departement, DEPARTEMENTS_AUTORISES, true)) {
        $erreurs['departement'] = "La valeur du département n'est pas autorisée";
    }

    // --- Si pas d'erreurs — on prépare les données nettoyées ---
    if (empty($erreurs)) {
        $succes = true;

        // On nettoie pour l'affichage — JAMAIS le mot de passe
        $dataValideNettoye = [
            'Prénom'      => htmlspecialchars($prenom,      ENT_QUOTES, 'UTF-8'),
            'Nom'         => htmlspecialchars($nom,         ENT_QUOTES, 'UTF-8'),
            'Email'       => htmlspecialchars($email,       ENT_QUOTES, 'UTF-8'),
            'Âge'         => $age,
            'Département' => htmlspecialchars($departement, ENT_QUOTES, 'UTF-8'),
        ];
    }
}
?>
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Inscription</title>
</head>
<body>

    <h1>Inscription</h1>

    <?php if ($succes) : ?>

        <!-- Affichage du succès -->
        <p style="color: green">Inscription réussie !</p>
        <h2>Vos informations :</h2>
        <ul>
            <?php foreach ($dataValideNettoye as $label => $valeur) : ?>
                <li><strong><?= $label ?> :</strong> <?= $valeur ?></li>
            <?php endforeach; ?>
        </ul>

    <?php else : ?>

        <!-- Affichage des erreurs AU-DESSUS du formulaire -->
        <?php if (!empty($erreurs)) : ?>
            <ul style="color: red">
                <?php foreach ($erreurs as $champ => $erreur) : ?>
                    <li><?= htmlspecialchars($erreur, ENT_QUOTES, 'UTF-8') ?></li>
                <?php endforeach; ?>
            </ul>
        <?php endif; ?>

        <!-- Formulaire -->
        <form method="POST" action="">

            <div>
                <label for="prenom">Prénom *</label><br>
                <input
                    type="text"
                    id="prenom"
                    name="prenom"
                    value="<?= htmlspecialchars($prenom, ENT_QUOTES, 'UTF-8') ?>"
                >
            </div>

            <div>
                <label for="nom">Nom *</label><br>
                <input
                    type="text"
                    id="nom"
                    name="nom"
                    value="<?= htmlspecialchars($nom, ENT_QUOTES, 'UTF-8') ?>"
                >
            </div>

            <div>
                <label for="email">Email *</label><br>
                <input
                    type="email"
                    id="email"
                    name="email"
                    value="<?= htmlspecialchars($email, ENT_QUOTES, 'UTF-8') ?>"
                >
            </div>

            <div>
                <label for="password">Mot de passe *</label><br>
                <!-- Jamais repopulé — value="" toujours vide -->
                <input
                    type="password"
                    id="password"
                    name="password"
                    value=""
                >
            </div>

            <div>
                <label for="age">Âge *</label><br>
                <input
                    type="number"
                    id="age"
                    name="age"
                    value="<?= htmlspecialchars((string)$age, ENT_QUOTES, 'UTF-8') ?>"
                    min="18"
                    max="65"
                >
            </div>

            <div>
                <label for="departement">Département *</label><br>
                <select name="departement" id="departement">
                    <option value="">-- Choisir --</option>
                    <?php foreach (DEPARTEMENTS_AUTORISES as $dept) : ?>
                    <option
                        value="<?= $dept ?>"
                        <?= $departement === $dept ? 'selected' : '' ?>
                    >
                        <?= ucfirst($dept) ?>
                    </option>
                    <?php endforeach; ?>
                </select>
            </div>

            <button type="submit">S'inscrire</button>

        </form>

    <?php endif; ?>

</body>
</html>
```

## À noter — le select repopulé dynamiquement

Tu avais les options codées en dur dans le HTML. Dans la correction, le select est généré depuis `DEPARTEMENTS_AUTORISES` — **une seule source de vérité** :

```php
// La liste est définie UNE FOIS en constante
const DEPARTEMENTS_AUTORISES = ['developpement', 'design', 'marketing', 'rh'];

// Le HTML est généré depuis cette constante
<?php foreach (DEPARTEMENTS_AUTORISES as $dept) : ?>
<option
    value="<?= $dept ?>"
    <?= $departement === $dept ? 'selected' : '' ?>  ← repopulation automatique
>
    <?= ucfirst($dept) ?>
</option>
<?php endforeach; ?>

// Si tu ajoutes un département → tu modifies juste la constante
// Le HTML se met à jour automatiquement ✅
```
