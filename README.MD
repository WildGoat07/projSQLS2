<!--
    Attention ! Il s'agit d'un fichier text en Markdown, pour voir un aperçu, allez ici -> https://github.com/WildGoat07/projSQLS2/blob/master/README.MD
-->
# Projet SQL S2

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
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT QN.no_question
    FROM question QN, theme T, rep_proposee RP
    WHERE libelle_theme="sport"
    AND etat_rep="TRUE"
    AND T.no_theme=QN.no_theme
    AND QN.no_question=RP.no_question
    GROUP BY QN.no_question
    HAVING COUNT (etat_rep)>=3
    ;
    CREATE TEMPORARY TABLE T2
    SELECT COUNT(*) n1
    FROM T1
    ;
    CREATE TEMPORARY TABLE T3
    SELECT COUNT(no_question) n2
    FROM question QN, theme T
    WHERE libelle_theme="sport"
    AND T.no_theme=QN.no_theme
    ;
    SELECT (100*n1)/n2
    FROM T2, T3
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
                session AS current_session
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
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT no_question,COUNT(*)AS n1
    FROM  session S, se_compose SC
    WHERE  S.no_quest=SC.no_quest
    GROUP BY no_question
    ;
    CREATE TEMPORARY TABLE T2
    SELECT SC.no_question,COUNT(*)AS n2
    FROM session S, rep_proposee RP, se_compose SC, rep_donnee RD
    WHERE RP.etat_rep="TRUE"
    AND RP.no_question=SC.no_question
    AND RD.no_ordre=RP.no_ordre
    AND RD.no_question=SC.no_question
    AND S.no_quest=SC.no_quest
    AND S.no_session=RD.no_session
    GROUP BY SC.no_question
    UNION
    SELECT no_question,0
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    AND no_question NOT IN 
    (
        SELECT RD.no_question
        FROM rep_proposee RP, rep_donnee RD
        WHERE RD.no_question=RP.no_question
        AND RD.no_ordre=RP.no_ordre
        AND etat_rep="TRUE"  
    )
    ;
    CREATE TEMPORARY TABLE T3
    SELECT SC.no_question,COUNT(*)AS n3
    FROM session S, rep_proposee RP, se_compose SC, rep_donnee RD
    WHERE RP.etat_rep="FALSE"
    AND RP.no_question=SC.no_question
    AND RD.no_ordre=RP.no_ordre
    AND RD.no_question=SC.no_question
    AND S.no_quest=SC.no_quest
    AND S.no_session=RD.no_session
    GROUP BY SC.no_question
    UNION
    SELECT no_question,0
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    AND no_question NOT IN 
    (
        SELECT RD.no_question
        FROM rep_proposee RP, rep_donnee RD
        WHERE RD.no_question=RP.no_question
        AND RD.no_ordre=RP.no_ordre
        AND etat_rep="FALSE"    
    )
    ;
    SELECT T1.no_question, 100*n2/n1 AS bonne_rep, 100*n3/n1 AS mauvais_rep, 100*(n1-n2-n3)/n1 AS false_rep
    FROM T1,T2,T3
    WHERE T1.no_question=T2.no_question
    AND T1.no_question=T3.no_question
    ORDER BY  T1.no_question
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
                session AS curr_session
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
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT no_session, count(no_question) n1,no_pers
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    GROUP BY no_session
    ;
    CREATE TEMPORARY TABLE T2
    SELECT S.no_session, count(RD.no_question) n2, no_pers
    FROM session S, rep_donnee RD, rep_proposee RP
    WHERE RD.no_session=S.no_session
    AND etat_rep= "TRUE"
    AND RP.no_question=RD.no_question
    AND RP.no_ordre=RD.no_ordre
    GROUP BY no_session
    ;
    SELECT DISTINCT T2.no_pers, nom_pers, prenom_pers, T2.no_session, n1,n2
    FROM T1, T2, personne P
    WHERE P.no_pers=T2.no_pers
    AND T1.no_session=T2.no_session
    AND n1=n2
    ORDER BY  nom_pers
    ```
1. > *"Quels utilisateurs ont répondu à des questionnaires portant sur tous les thèmes pour lesquels il existe des questionnaires ?"*
    ```sql
    SELECT
        -- formattage....
        CONCAT(personne.nom_pers, " ", personne.prenom_pers) AS Personnes
    FROM
        personne
        NATURAL JOIN session
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
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT P.no_pers, COUNT(DISTINCT Q.no_theme) n1
    FROM  personne P, session S, questionnaire Q
    WHERE P.no_pers=S.no_pers
    AND S.no_quest=Q.no_quest
    GROUP BY no_pers
    ;

    CREATE TEMPORARY TABLE T2
    SELECT COUNT(T.no_theme) n2
    FROM theme T, questionnaire QN
    Where T.no_theme=QN.no_theme
    ;

    SELECT P.no_pers, nom_pers, prenom_pers, n1, n2
    FROM T1,T2,personne P
    WHERE P.no_pers=T1.no_pers
    AND n1=n2
    ```
1. > *"Quels utilisateurs ont répondu au moins une fois à tous les questionnaires ?"*
    ```sql
    -- approche arithmétique
    SELECT
        CONCAT(UPPER(personne.nom_pers), " ", personne.prenom_pers) AS Personnes
    FROM
        personne
        NATURAL JOIN session
    GROUP BY personne.no_pers
    HAVING COUNT(DISTINCT session.no_quest) =
    (
        SELECT
            COUNT(*)
        FROM
            questionnaire
    )
    ```
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT P.no_pers, nom_pers, prenom_pers, COUNT(DISTINCT  S.no_quest) n1
    FROM personne P, session S, questionnaire Q
    WHERE P.no_pers=S.no_pers
    AND S.no_quest=Q.no_quest
    GROUP BY no_pers
    ;
    CREATE TEMPORARY TABLE T2
    SELECT COUNT(no_quest) n2
    FROM questionnaire
    ;
    Select P.no_pers, P.nom_pers, P.prenom_pers 
    From T1, T2, personne P
    Where P.no_pers=T1.no_pers
    And n1=n2
    ```
1. > *"Score de la session numéro 23."*
    ```sql
    SELECT
        bonnes_reponses.val AS `Réponses justes`, mauvaises_reponses.val AS `Réponses fausses`, non_reponses.val AS `Non-réponses`, bonnes_reponses.val * 2 - mauvaises_reponses.val * 2 - non_reponses.val AS Score
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
        ) AS mauvaises_reponses,
        (
            SELECT
            	COUNT(*) as val
            FROM
            	session
            	NATURAL JOIN se_compose
            	LEFT JOIN rep_donnee USING (no_session, no_question)
            WHERE
            	rep_donnee.no_ordre IS null
            	AND session.no_session = 18
        ) AS non_reponses
    ```
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT S.no_session, COUNT(RD.no_question) n1
    FROM session S, rep_donnee RD, rep_proposee RP
    WHERE S.no_session="10"
    AND S.no_session=RD.no_session
    AND RD.no_question=RP.no_question
    AND RD.no_ordre=RP.no_ordre
    AND etat_rep="TRUE"
    ;

    CREATE TEMPORARY TABLE T2
    SELECT S.no_session, COUNT(RD.no_question) n2
    FROM session S, rep_donnee RD, rep_proposee RP
    WHERE S.no_session="10"
    AND S.no_session=RD.no_session
    AND RD.no_question=RP.no_question
    AND RD.no_ordre=RP.no_ordre
    AND etat_rep="FALSE"
    ;

    CREATE TEMPORARY TABLE T3
    SELECT S.no_session, COUNT(SC.no_question) n3
    FROM se_compose SC, session S
    WHERE S.no_session="10"
    AND S.no_quest=SC.no_quest
    AND SC.no_question NOT IN(
                SELECT RD.no_question
                FROM session S, rep_donnee RD
                WHERE S.no_session="10"
                AND S.no_session=RD.no_session )
    ;

    SELECT T1.no_session, 2*n1-2*n2-n3 AS score
    FROM   T1, T2, T3
    ```
1. > *"Quel est le pourcentage des personnes ayant participé à au moins 1 session qui ont obtenu un score supérieur à 0 lors d’au moins 1 session ?"*
    ```sql
    SELECT
        pers.val / total.val AS Pourcentage
    FROM
        (
            -- le total des personnes ayant joué
            SELECT
                COUNT(DISTINCT session.no_pers) AS val
            FROM
                session
        ) AS total,
        (
            SELECT
                COUNT(DISTINCT curr_session.no_pers) AS val
            FROM
                session AS curr_session
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
                ) * 2 -
            	(
                    -- les non-réponses
                    SELECT
                    	COUNT(*)
                    FROM
                    	session
                    	NATURAL JOIN se_compose
                    	-- on a récupéré toutes les questions du questionnaire en cours
                    	LEFT JOIN rep_donnee USING(no_session, no_question) -- on récupère les réponses données
                    WHERE
                    	session.no_session = curr_session.no_session
                    	AND rep_donnee.no_ordre IS null
                    
                )
        ) AS pers
    ```
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT S.no_session, SC.no_question,COUNT(*)AS n1
    FROM  session S, se_compose SC
    WHERE  S.no_quest=SC.no_quest
    GROUP BY no_session
    ;
    CREATE TEMPORARY TABLE T2
    SELECT  S.no_session, RD.no_question,COUNT(distinct RD.no_question)AS n2
    FROM session S, rep_proposee RP, rep_donnee RD
    WHERE S.no_session=RD.no_session
    AND RD.no_ordre=RP.no_ordre
    AND RD.no_question=RP.no_question
    AND etat_rep="TRUE"
    GROUP BY S.no_session
    UNION
    SELECT  S.no_session, SC.no_question,0
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    AND no_question NOT IN 
    (
        SELECT RD.no_question
        FROM rep_proposee RP, rep_donnee RD
        WHERE RD.no_question=RP.no_question
        AND RD.no_ordre=RP.no_ordre
        AND etat_rep="TRUE"  
    )
    ;
    CREATE TEMPORARY TABLE T3
    SELECT  S.no_session, RD.no_question,COUNT(distinct RD.no_question)AS n3
    FROM session S, rep_proposee RP, rep_donnee RD
    WHERE S.no_session=RD.no_session
    AND RD.no_ordre=RP.no_ordre
    AND RD.no_question=RP.no_question
    AND etat_rep="FALSE"
    GROUP BY S.no_session
    UNION
    SELECT  S.no_session, SC.no_question,0
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    AND no_question NOT IN 
    (
        SELECT RD.no_question
        FROM rep_proposee RP, rep_donnee RD
        WHERE RD.no_question=RP.no_question
        AND RD.no_ordre=RP.no_ordre
        AND etat_rep="FALSE"  
    )
    ;
    CREATE TEMPORARY TABLE T4
    SELECT T1.no_session, n1 AS total_rep, n2 AS bon_rep, n3 AS mauvai_rep, n1-n2-n3 AS false_rep, 3*n2-n3-n1 AS score
    FROM T1,T2,T3
    WHERE T1.no_session=T2.no_session
    AND T1.no_session=T3.no_session
    GROUP BY  T1.no_session
    ;
    CREATE TEMPORARY TABLE T5
    SELECT T4.no_session, S.no_pers, P.nom_pers, P.prenom_pers, COUNT(DISTINCT S.no_pers) rp
    FROM T4, session S, personne P
    WHERE score>0
    AND T4.no_session=S.no_session
    AND S.no_pers=P.no_pers
    ;
    CREATE TEMPORARY TABLE T6
    SELECT COUNT(DISTINCT S.no_pers) rt
    FROM personne P, session S
    WHERE P.no_pers=S.no_pers
    ;
    SELECT 100*rp/rt
    FROM T5,T6
    ```
1. > *"Quelles personnes ont réalisés les 10 meilleurs scores ?"*
    ```sql
    SELECT
        CONCAT(UPPER(personne.nom_pers), " ", personne.prenom_pers) AS Personnes,
        (
            -- le nombre de bonnes réponses
            SELECT
            	COUNT(*)
            FROM
            	rep_donnee
            	NATURAL JOIN rep_proposee
            WHERE
            	rep_donnee.no_session = curr_session.no_session
            	AND rep_proposee.etat_rep = true
        ) * 2 -
        (
            -- les mauvaises réponses
            SELECT
            	COUNT(*)
            FROM
            	rep_donnee
            	NATURAL JOIN rep_proposee
            WHERE
            	rep_donnee.no_session = curr_session.no_session
            AND rep_proposee.etat_rep = false
        ) * 2 -
        (
            -- les non-réponses
            SELECT
            	COUNT(*)
            FROM
            	session
            	NATURAL JOIN se_compose
            	LEFT JOIN rep_donnee USING (no_session, no_question)
            WHERE
            	session.no_session = curr_session.no_session
            	AND rep_donnee.no_ordre IS null
        ) AS Score, questionnaire.titres_quest AS Questionnaire
    FROM
        personne
        NATURAL JOIN session AS curr_session
        NATURAL JOIN questionnaire
    ORDER BY Score DESC
    LIMIT 10
    ```
    alternative :
    ```sql
    CREATE TEMPORARY TABLE T1
    SELECT S.no_session, SC.no_question,COUNT(*)AS n1
    FROM  session S, se_compose SC
    WHERE  S.no_quest=SC.no_quest
    GROUP BY no_session
    ;
    CREATE TEMPORARY TABLE T2
    SELECT  S.no_session, RD.no_question,COUNT(distinct RD.no_question)AS n2
    FROM session S, rep_proposee RP, rep_donnee RD
    WHERE S.no_session=RD.no_session
    AND RD.no_ordre=RP.no_ordre
    AND RD.no_question=RP.no_question
    AND etat_rep="TRUE"
    GROUP BY S.no_session
    UNION
    SELECT  S.no_session, SC.no_question,0
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    AND no_question NOT IN 
    (
        SELECT RD.no_question
        FROM rep_proposee RP, rep_donnee RD
        WHERE RD.no_question=RP.no_question
        AND RD.no_ordre=RP.no_ordre
        AND etat_rep="TRUE"  
    )
    ;
    CREATE TEMPORARY TABLE T3
    SELECT  S.no_session, RD.no_question,COUNT(distinct RD.no_question)AS n3
    FROM session S, rep_proposee RP, rep_donnee RD
    WHERE S.no_session=RD.no_session
    AND RD.no_ordre=RP.no_ordre
    AND RD.no_question=RP.no_question
    AND etat_rep="FALSE"
    GROUP BY S.no_session
    UNION
    SELECT  S.no_session, SC.no_question,0
    FROM session S, se_compose SC
    WHERE S.no_quest=SC.no_quest
    AND no_question NOT IN 
    (
        SELECT RD.no_question
        FROM rep_proposee RP, rep_donnee RD
        WHERE RD.no_question=RP.no_question
        AND RD.no_ordre=RP.no_ordre
        AND etat_rep="FALSE"  
    )
    ;
    CREATE TEMPORARY TABLE T4
    SELECT T1.no_session, n1 AS total_rep, n2 AS bon_rep, n3 AS mauvai_rep, n1-n2-n3 AS false_rep, 3*n2-n3-n1 AS score
    FROM T1,T2,T3
    WHERE T1.no_session=T2.no_session
    AND T1.no_session=T3.no_session
    GROUP BY  T1.no_session
    ORDER BY score DESC
    LIMIT 10
    ;
    SELECT T4.no_session, P.no_pers, P.nom_pers, P.prenom_pers, score
    FROM T4, session S, personne P
    WHERE T4.no_session=S.no_session
    AND S.no_pers=P.no_pers
    ```
1. > *"Existe-t-il des réponses données enregistrées ne correspondant pas à une question posée ?"*
    ```sql
    SELECT
        curr_rep.no_session AS `Session`, curr_rep.no_question AS `ID de la question répondu`
    FROM
        rep_donnee AS curr_rep
    WHERE
        -- pour chaque réponse, on vérifie qu'elle ne soit pas dans les questions posées lors de la session
        curr_rep.no_question NOT IN
        (
            SELECT
                se_compose.no_question
            FROM
                session
                NATURAL JOIN questionnaire
                NATURAL JOIN se_compose
            WHERE
                session.no_session = curr_rep.no_session
        )
    ```
    alternative :
    ```sql
    SELECT RD.no_session, RD.no_question
    FROM rep_donnee RD
    WHERE RD.no_question NOT IN(
                SELECT SC.no_question
                FROM session S, se_compose SC
                WHERE S.no_quest=SC.no_quest
                )
    ```
