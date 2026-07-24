---
name: recherche-appart
description: Cherche sur le web les appartements à vendre dans le 2e arrondissement de Paris de moins de 20 m², et met à jour annonces.xlsx en marquant les nouvelles trouvailles. À utiliser quand l'utilisateur demande de chercher/rafraîchir/mettre à jour les annonces d'appartement.
---

# Recherche d'appartements — Paris 2e, moins de 20 m²

## Critères (figés)

| Critère | Valeur |
|---|---|
| Transaction | Achat (vente) |
| Localisation | Paris 2e arrondissement (75002) |
| Surface | Strictement inférieure à 20 m² |
| Prix | Pas de plafond |

Le marché est minuscule — quelques biens à un instant donné, parfois un seul.
Trouver 1 à 3 annonces est un résultat normal et suffisant. Ne pas élargir les
critères pour gonfler le résultat : un bien de 21 m² ou situé dans le 3e n'a
pas sa place dans le fichier.

Attention au piège de surface : beaucoup d'annonces de « petites surfaces »
tournent autour de 22-25 m². Le seuil est **strictement sous 20 m²** — vérifier
la surface Carrez avant de retenir un bien, c'est le filtre le plus souvent
raté par les moteurs de recherche.

## Procédure

### 1. Lire l'existant

```bash
python3 .claude/skills/recherche-appart/update_xlsx.py --list
```

Renvoie les liens déjà présents dans `annonces.xlsx` (liste vide si le fichier
n'existe pas encore). Sert à ne pas ré-annoncer un bien déjà connu.

### 2. Chercher

Lancer plusieurs `WebSearch` en parallèle, avec des formulations variées — les
moteurs ne renvoient pas les mêmes résultats selon le vocabulaire :

- `studio à vendre Paris 75002 15m2`
- `petit studio à vendre Paris 2e arrondissement 18m2`
- `vente studio Paris 2ème arrondissement Montorgueil Sentier Bonne Nouvelle`
- `chambre de bonne à vendre Paris 2e arrondissement`
- `studio 12m2 vendre Paris 2eme`

Puis `WebFetch` sur les pages de résultats et les annonces qui paraissent
correspondre, pour en extraire les détails.

**Ne jamais contourner une protection.** Si un site renvoie 403, bloque via
`robots.txt`, ou exige une authentification, passer au suivant et le noter dans
le rapport final. Leboncoin et SeLoger sont normalement inaccessibles par ce
chemin — c'est attendu, ce n'est pas une erreur à corriger.

Sites généralement exploitables : Bien'ici, PAP, immobilier.notaires.fr,
Logic-Immo, Figaro Immobilier, sites d'agences du quartier.

### 3. Extraire

Pour chaque bien retenu, relever ce qui est disponible :

`prix`, `surface_m2`, `pieces`, `etage`, `ascenseur`, `dpe`,
`charges_mensuelles`, `rue_ou_quartier`, `source`, `lien`, `description`
(une phrase, avec les points saillants : travaux à prévoir, dernier étage,
sans vis-à-vis, cave, etc.)

Règles :
- **Ne rien inventer.** Un champ absent de l'annonce reste vide.
- Le prix au m² est calculé par le script, ne pas le fournir.

### 4. Écrire

Passer les annonces en JSON sur stdin :

```bash
echo '[{"prix": 195000, "surface_m2": 18, "pieces": 1, "etage": "3e",
"ascenseur": "non", "dpe": "E", "charges_mensuelles": 95,
"rue_ou_quartier": "rue Saint-Sauveur", "source": "Bien'"'"'ici",
"lien": "https://...", "description": "Studio refait, dernier étage, sans vis-à-vis"}]' \
  | python3 .claude/skills/recherche-appart/update_xlsx.py
```

Le script fusionne par `lien` : il ajoute les nouveautés, marque la colonne
`Nouveau`, calcule le prix au m², et **ne touche pas** aux lignes existantes ni
à la colonne `Notes` (celle-ci appartient à l'utilisateur).

### 5. Rapporter

Résumer en quelques lignes :
- Combien de nouvelles annonces, combien déjà connues
- Pour chaque nouveauté : prix, surface, prix/m², rue, et ce qui la distingue
- Les sites qui ont bloqué l'accès, s'il y en a

Repère de marché pour situer un prix/m² : le 2e arrondissement tourne autour
de 10 000 à 12 000 €/m². Le signaler quand un bien s'en écarte nettement.
