<!--
    Attention ! Il s'agit d'un fichier text en markdown, pour voir un aperçu, allez ici -> https://github.com/WildGoat07/projSQLS2/blob/master/README.MD
-->
# Projet SQL S2

Note : Notre table `session` s'appelle `quest_session` car `session` est un mot-clé. Pour utiliser les requêtes suivantes, il faudra 

## Requêtes :

1. > *"Quel pourcentage des questions portant sur le sport comporte au moins 3 bonnes réponses proposées ?"*
    ```sql
    SELECT
        -- ici on récupère le nombre de questions qui ont 3 réponses justes
        COUNT(*) /
        (
            -- ici on récupère le nombre total de questions du thème choisi
            SELECT
                COUNT(*)
            FROM
                question
                NATURAL JOIN theme
            WHERE
                theme.libelle_theme = "Sport"
        ) AS pourcentage
    FROM
        (
        -- cette requete récolte les questions qui ont 3 réponses juste au thème donné
        SELECT
            *
        FROM
            question
            NATURAL JOIN theme
            NATURAL JOIN rep_proposee
        WHERE
            theme.libelle_theme = "Sport"
            AND rep_proposee.etat_rep = TRUE
        GROUP BY
            question.libelle
        HAVING
            COUNT(*) >= 3
        ) AS t; -- on doit obligatoirement donner un alias à SQL pour les SELECT dans un FROM ¯\_(ツ)_/¯
    ```
1. > *"Quels sont, pour chaque question, les pourcentages de réponses données correctes, deréponses données fausses, et de non-réponses ?"*
    ```sql
    SELECT
        question.no_question AS numero,
        question.libelle AS libelle,
        (
            -- cette requête récupère le nombre de réponses justes données
            SELECT
                COUNT(*)
            FROM
                rep_donnee
                NATURAL JOIN rep_proposee
            WHERE
                rep_donnee.no_question = numero
                AND rep_proposee.etat_rep = true
        ) /
        (
            -- cette requête récupère le nombre de réponses données
            SELECT
                COUNT(*)
            FROM
                rep_donnee
                NATURAL JOIN rep_proposee
            WHERE
                rep_donnee.no_question = numero
        ) AS "bonne réponses",
        (
            -- cette requête récupère le nombre de réponses fausses données
            SELECT
                COUNT(*)
            FROM
                rep_donnee
                NATURAL JOIN rep_proposee
            WHERE
                rep_donnee.no_question = numero
                AND rep_proposee.etat_rep = false
        ) /
        (
            -- cette requête récupère le nombre de réponses données
            SELECT
                COUNT(*)
            FROM
                rep_donnee
                NATURAL JOIN rep_proposee
            WHERE
                rep_donnee.no_question = numero
        ) AS "mauvaises réponses",
        -- on fait "1-pourcentage" car sinon, on renvoie le pourcentage de réponses données, or on veut les non-données
        1 - (
            -- cette requête donne le nombre de réponses données
            SELECT
                COUNT(*)
            FROM
                rep_donnee
            WHERE
                rep_donnee.no_question = numero
        ) /
        (
            -- cette requête donne le nombre de fois que la question actuelle a été demandée
            SELECT
                COUNT(*)
            FROM
                quest_session AS current_session
            WHERE
                numero IN
                (
                    -- cette requête donne la liste des questions du questionnaire de la session en cours
                    SELECT
                        se_compose.no_question
                    FROM
                        se_compose
                    WHERE current_session.no_quest = se_compose.no_quest
                )
        ) AS "non-réponses"
    FROM
        question;
    ```
1. > *"Liste alphabétique des personnes ayant donné une réponse correcte à toutes les questions lors d’au moins une session."*
    ```sql
    SELECT
        -- formattage....
        CONCAT(UPPER(curr_pers.nom_pers), " ", curr_pers.prenom_pers) as Personnes
    FROM
        personne AS curr_pers
    WHERE
        -- on sélectionne toute les personne où cette sous requête a trouvé une session où toutes les réponses sont justes
        EXISTS
        (
            -- cette sous-requête donne toutes les sessions avec 100% de réponses justes
            SELECT
                curr_session.no_session
            FROM
                quest_session AS curr_session
                NATURAL JOIN questionnaire AS curr_questionnaire
            WHERE
                -- on lie à notre personne actuelle
                curr_session.no_pers = curr_pers.no_pers
                -- on vérifie qu'il n'y a aucune réponse non validée
                AND NOT EXISTS
                (
                    -- cette requête sélectionne toutes les réponses fausse ou non-répondues
                    SELECT
                        rep_donnee.no_question
                    FROM
                        rep_donnee
                        NATURAL JOIN rep_proposee
                        -- on fait un RIGHT JOIN car on veut garder les questions où on n'a pas de réponses (null)
                        RIGHT JOIN se_compose USING(no_question)
                    WHERE
                        -- on se lie au questionnaire actuel de notre session
                        se_compose.no_quest = curr_questionnaire.no_quest
                        AND
                        -- s'il s'agit d'une réponse donnée à la bonne session et qu'elle est fausse
                        ( ( rep_donnee.no_session = curr_session.no_session AND rep_proposee.etat_rep = false)
                        -- ou bien que cette réponse est absente
                            OR rep_donnee.no_session IS null)
                )
        )
    -- trie par rapport au nom et au prénom
    ORDER BY curr_pers.nom_pers, curr_pers.prenom_pers;
    ```
1. > *"Quels utilisateurs ont répondu à des questionnaires portant sur tous les thèmes pour lesquels il existe des questionnaires ?"*
    ```sql
    SELECT
        -- formattage....
        CONCAT(personne.nom_pers, " ", personne.prenom_pers) AS Personnes
    FROM
        personne
        NATURAL JOIN quest_session
        NATURAL JOIN questionnaire
    -- on sélectionne tous les questionnaires répondus pour chaque personne
    GROUP BY personne.no_pers
    -- si on a autant de thèmes répondus différents qu'il existe de thèmes différents
    HAVING COUNT(DISTINCT questionnaire.no_theme) = 
        (
            SELECT
                -- le nombre de thèmes différents possibles à répondre
                COUNT(DISTINCT questionnaire.no_theme)
            FROM
                questionnaire
        );
    ```
1. > *"Quels utilisateurs ont répondu au moins une fois à tous les questionnaires ?"*
    ```sql
    -- approche arithmétique
    SELECT
        CONCAT(UPPER(personne.nom_pers), " ", personne.prenom_pers) AS Personnes
    FROM
        personne
        NATURAL JOIN quest_session
    GROUP BY personne.no_pers
    HAVING COUNT(DISTINCT quest_session.no_quest) =
    (
        SELECT
            COUNT(*)
        FROM
            questionnaire
    )
    ```
1. > *"Score de la session numéro 23."*
    
    **Note** : On va mettre comme règle : une bonne réponse = 2 pts, une mauvaise réponse = -1 pt, une non-réponse = 0 pts pour pimenter les choses 🔥
    ```sql
    SELECT
        bonnes_reponses.val AS `Réponses justes`, mauvaises_reponses.val AS `Réponses fausses`, bonnes_reponses.val * 2 - mauvaises_reponses.val AS Score
    FROM
        (
            SELECT
                COUNT(*) AS val
            FROM
                rep_donnee
                NATURAL JOIN rep_proposee
            WHERE
                rep_donnee.no_session = 18
                AND rep_proposee.etat_rep = true
        ) AS bonnes_reponses,
        (
            SELECT
                COUNT(*) AS val
            FROM
                rep_donnee
                NATURAL JOIN rep_proposee
            WHERE
                rep_donnee.no_session = 18
                AND rep_proposee.etat_rep = false
        ) AS mauvaises_reponses
    ```
1. > *"Quel est le pourcentage des personnes ayant participé à au moins 1 session qui ont obtenu un score supérieur à 0 lors d’au moins 1 session ?"*

    Pareil, on calcul le score de la même manière qu'à la question précédente
    ```sql
    SELECT
        pers.val / total.val AS Pourcentage
    FROM
        (
            -- le total des personnes ayant joué
            SELECT
                COUNT(DISTINCT quest_session.no_pers) AS val
            FROM
                quest_session
        ) AS total,
        (
            SELECT
                COUNT(DISTINCT curr_session.no_pers) AS val
            FROM
                quest_session AS curr_session
            WHERE
            -- calcul du score (copie pasta de la question précédente)
            0 < (
                    -- le nombre de bonnes réponses
                    SELECT
                        COUNT(*)
                    FROM
                        rep_donnee
                        NATURAL JOIN rep_proposee
                    WHERE
                        rep_donnee.no_session = curr_session.no_session
                        AND rep_proposee.etat_rep = true
                ) * 2 -  -- on multiplie par deux, et on enlève le nombre de mauvaises réponses pour le score
                (
                    SELECT
                        COUNT(*)
                    FROM
                        rep_donnee
                        NATURAL JOIN rep_proposee
                    WHERE
                        rep_donnee.no_session = curr_session.no_session
                        AND rep_proposee.etat_rep = false
                )
        ) AS pers
    ```
1. > *"Quelles personnes ont réalisés les 10 meilleurs scores ?"*
    ```sql
    ```
1. > *"Existe-t-il des réponses données enregistrées ne correspondant pas à une question posée ?"*
    ```sql
    ```