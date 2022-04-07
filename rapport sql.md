# Rapport mini projet SQL
identifiants Oracle :

- Louis : i2a06b
- Julien : i2a07a
- Martin : i2a02b
- Léo : i2a02a
- Emerik : i2a04a

## Table Recharge
Recharge correspond à une table temporaire dans laquelle les erreurs sont corrigées. Au lieu de créer des nouvelles tables à partir de basetd rechargeelectrique on le fait à partir de Recharge pour ne pas copier les erreurs. 

### Création de la table
Pour créer cette table, on prend tout le contenu de basetd rechargeelectrique.
```sql
create table recharge as select * from basetd.rechargeelectrique; 
```
### Correction des erreurs
La table recharge possède plusieurs erreurs. 
Avec ces requêtes, on corrige :

- les doublons de communes (deux possèdent le code insee 85234 et deux autres 85223)
- des stations situées en Vendée alors qu'elles sont inscrites à la Réunion dans recharge
- une station située en Gironde alors qu'elle devrait être en Loire-Atlantique
- l'écriture des horaires inconsistante 
```sql
UPDATE recharge 
SET
    commune = 'Sainte-Hermine'
WHERE
    commune = 'Saint-Martin-Lars-en-Sainte-Hermine';

UPDATE recharge
SET
    commune = 'Saint-Jean-de-Monts'
WHERE
    insee = '85234.0';

UPDATE recharge
SET
    département = 'Vendée'
WHERE
    département = 'La Réunion';

UPDATE recharge
SET
    commune = 'Rezé',
    département = 'Loire-Atlantique'
WHERE
    département = 'Gironde';
    
UPDATE recharge
SET
    département = 'Vendée'
WHERE
    département = 'Ille-et-Vilaine';

UPDATE recharge
SET
    horaires = '24/24 7/7';
```

### Donner les droits
Pour que les utilisateurs créent les autres tables, il faut leur donner le droit de sélectionner la table recharge.
```sql
grant select on recharge to i2a07a;
grant select on recharge to i2a02b;
grant select on recharge to i2a02a;
grant select on recharge to i2a04a;
```
## Table Commune
La table commune possède toutes les communes dans les Pays de la Loire avec leur code insee, le nom de commune et le numéro de leur département.
### Création de la table
```sql
create table commune as
select distinct insee, commune as nomcommune, code_dep as numdep from recharge;
```
### Contraintes d'intégrité en local
La seule contrainte d'intégrité sur commune en local est la clé primaire, le code insee.
```sql
alter table commune add constraint Commune_PK primary key (insee);
```

### Contraintes d'intégrité distantes
L'attribut numdep de la table commune doit faire référence à la clé primaire de la table departement. Il est nécessaire d'obtenir l'autorisation de référencer cet attribut avant d'ajouter la contrainte (voir les droits donnés lors de la création de la table département). 
```sql
alter table commune add constraint Commune_FK foreign key (numdep) references departement(numdep);
```
### Donner les droits
On donne les droits de sélectionner la table commune aux autres utilisateurs. Le "with grant option" leur permet de donner le droit de sélectionner sur des vues qu'ils ont créé à partir de la table commune. 

On donne également le droit de référencer l'attribut insee au possesseur de la table Station.
```sql
grant select on commune to i2a07a with grant option;
grant select on commune to i2a02b with grant option;
grant select on commune to i2a02a with grant option;
grant select on commune to i2a04a with grant option;

GRANT REFERENCES(insee) ON commune to i2a04a;
```

## Table Département
La table departement donne la liste des départements dans les Pays de la Loire avec le code de département et le nom de département. 
### Création de la table
```sql
create table departement as
select distinct code_dep as numdep, département as nomdep from i2a06b.recharge;
```

### Contraintes d'intégrité en local
La seule contrainte d'intégrité en local à ajouter est la clé primaire, le code de département. 
```sql
alter table departement add constraint Departement_PK primary key (numdep);
```
### Contraintes d'intégrité distantes
Il n'y a pas de contraintes d'intégrité distantes dans la table departement. 
### Donner les droits
On donne les droits de sélectionner la table departement aux autres utilisateurs.

On donne également le droit de réferencer l'attribut numdep au possesseur de la table commune pour relier les communes à un département. 
```sql
grant select on departement to i2a07a;
grant select on departement to i2a02b;
grant select on departement to i2a06b;
grant select on departement to i2a04a;

GRANT REFERENCES(numdep) ON departement to i2a06b;
```
## Table Station
La table station donne la liste des stations des pays de la loire avec le code de la station, son libellé, son addresse, son code insee, les observations (notes sur la station, par exemple le type de recharge), la date de mise a jour, sa localisation en coordonnées geographique, le nombre de points de charge, la puissance maximum des points de charge, le type de prise, les horaires, si l'accès est gratuit ou payant, et enfin l'aménageur.
### Création de la table
```sql
CREATE TABLE STATION AS
SELECT CODESTATION, LIBELLÉSTATION, ADRESSE, INSEE, OBSERVATIONS, DATEDEMISEAJOUR, LOCALISATION, CAST(REPLACE(NOMBREPOINTSDECHARGE, '.0', '') AS INT) AS NBPOINTSDECHARGE, TO_NUMBER(PUISSANCEMAXIMUM, '999.99') AS PUISSANCEMAXIMUM, TYPEDEPRISE, HORAIRES, ACCÈSRECHARGE, AMÉNAGEUR AS AMENAGEUR
FROM I2A06B.RECHARGE;
```
### Contraintes d'intégrité en local
Il faut ajouter la clé primaire, le code de la station :
```sql
ALTER TABLE STATION ADD CONSTRAINT CODESTATIONPK PRIMARY KEY(CODESTATION);
```
### Contraintes d'intégrité distantes
On ajoute ensuite les clés étrangères, le code insee qui référence celui présent dans la table commune et l'aménageur qui référence celui présent dans la table intervenant.
```sql
ALTER TABLE STATION ADD CONSTRAINT INSEEFK FOREIGN KEY(INSEE) REFERENCES I2A06B.COMMUNE(INSEE);
ALTER TABLE STATION ADD CONSTRAINT AMENAGEURFK FOREIGN KEY(AMENAGEUR) REFERENCES I2A02B.INTERVENANT(AMENAGEUR);
```
### Donner les droits
On donne les droits de sélectionner la table station aux autres utilisateurs. 

Le "with grant option" leur permet de donner le droit de sélectionner sur des vues qu'ils ont créé à partir de la table station. 
```sql
GRANT SELECT ON STATION TO I2A06B WITH GRANT OPTION; 
GRANT SELECT ON STATION TO I2A07A WITH GRANT OPTION; 
GRANT SELECT ON STATION TO I2A02B WITH GRANT OPTION; 
GRANT SELECT ON STATION TO I2A02A WITH GRANT OPTION; 
```


## Table Intervenant
La table intervenant liste tous les aménageurs avec les opérateurs et l'enseigne. 
Chaque aménageur possède toujours le même opérateur avec la même enseigne.
### Création de la table
```sql
create table intervenant as
select distinct aménageur as amenageur, enseigne as enseigne, opérateur as operateur from recharge;
```
### Contraintes d'intégrité en local
La seule contrainte d'intégrité en local de la table intervenant est la clé primaire, amenageur. 
```sql
alter table intervenant add constraint Intervant_PK primary key (amenageur);
```
### Contraintes d'intégrité distantes
Il n'y a pas de contraintes d'intégrité distantes dans la table intervenant. 
### Donner les droits
On donne les droits aux autres utilisateurs de select sur la table intervenant.

On donne également le droit de référencer l'attribut amenageur au possesseur de la table station, nécessaire pour donner l'aménageur de chaque station électrique. 
```sql
GRANT SELECT ON INTERVENANT TO i2a06b;
GRANT SELECT ON INTERVENANT TO i2a07a;
GRANT SELECT ON INTERVENANT TO i2a02a;
GRANT SELECT ON INTERVENANT TO I2A04A;
GRANT REFERENCES(amenageur) ON INTERVENANT TO I2A04A;
```
## Requêtes

### Affichez pour chaque Aménageur le nombre de prises par département. 
```sql
/* Sarthe 72 */
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 72
group by s.amenageur, co.nomcommune;
/* Maine-et-Loire 49 */
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 49
group by s.amenageur, co.nomcommune;
/* Vendée 85 */
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 85
group by s.amenageur, co.nomcommune;
/* Loire-Atlantique 44 */
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 44
group by s.amenageur, co.nomcommune;
/* Mayenne 53 */
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 53
group by s.amenageur, co.nomcommune;
```
### Afficher pour chaque Aménageur le nombre de prises avec la puissance Max 
```sql
select inter.amenageur,
    (select sum(s.nbpointsdecharge) from i2a04a.station s where s.amenageur = inter.amenageur) as nbpointsdecharge,
    (select max(puissancemaximum) from i2a04a.station s where s.amenageur = inter.amenageur) as puissancemaximum
from i2a02b.intervenant inter;
```
### Affichez pour chaque département la répartition des prises. 
```sql
select dep.nomdep,
    (select sum(s.nbpointsdecharge) 
        from i2a04a.station s
        where s.insee IN (select c.insee from i2a06b.commune c where c.numdep = dep.numdep)) as nbpointsdecharge
from i2a02a.departement dep;
```
## Proposez 2 autres types de requêtes 

### Pour chaque aménageur, donner la station mise à jour la plus récemment
```sql
select t1.amenageur, s.codestation, s.datedemiseajour from (
select inter.amenageur,
    (select codestation
        from (
            select * 
                from i2a04a.station s
            order by datedemiseajour DESC
        ) 
         where rownum <= 1) as codestation
from i2a02b.intervenant inter) t1, i2a04a.station s
where t1.codestation = s.codestation;
```

### Pour chaque commune, donner le nombre de points de charge gratuits
```sql
select co.nomcommune,
    (select count(*) from i2a04a.station s where lower(s.accèsrecharge) = 'gratuit' and co.insee = s.insee) as nbpointsdechargegratuit
from i2a06b.commune co
order by 1;
```

## Synthèse 
### Créer les vues
On créé les vues à partir des requêtes précedentes.
```sql
/* Sarthe 72 i2a02a */
create or replace view VSarthe as
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 72
group by s.amenageur, co.nomcommune;
/* Maine-et-Loire 49 i2a02b */
create or replace view VMaine as 
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 49
group by s.amenageur, co.nomcommune;
/* Vendée 85 i2a04a */
create or replace view VVendee as
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 85
group by s.amenageur, co.nomcommune;
/* Loire-Atlantique 44 i2a06b */
create or replace view VLA as
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 44
group by s.amenageur, co.nomcommune;
/* Mayenne 53 i2a06b */
create or replace view VMayenne as
select s.amenageur, co.nomcommune, sum(s.nbpointsdecharge) as nbpointsdecharge
from i2a04a.station s, i2a06b.commune co
where s.insee = co.insee
    and co.numdep = 53
group by s.amenageur, co.nomcommune;
```

## Donner les droits
Le possesseur de commune doit donner les droits de sélectionner cette table aux autres utilisateurs pour qu'ils puissent créer des vues de commune. Le grant option leur permet de donner le droit de sélectionner leurs vues à l'étudiant qui crée la vue global. 
```sql
/* i2a06b */
grant select on commune to i2a02a with grant option;
grant select on commune to i2a04a with grant option;
grant select on commune to i2a02b with grant option;
```
Le possesseur de station donne les doits de sélectionner sa table avec grant option pour que les autres utilisateurs créent des vues et donnent le droit de la sélectionner. 
```sql
/* i2a04a */
grant select on station to i2a02a with grant option;
grant select on station to i2a06b with grant option;
grant select on station to i2a02b with grant option;
grant select on VVendee to i2a06b with grant option;
```
Les possesseurs de VSarthe et VMaine doivent donner les droits de sélectionner leur vue respective à l'utilisateur qui crée la vue globale. 
```sql
/* i2a02a */
grant select on VSarthe to i2a06b with grant option;

/* i2a02b */
grant select on VMaine to i2a06b with grant option;
```

### Créer la vue globale
Pour faire la vue globale, on fait l'union de la vue des différents utilisateurs. Les droits sont déjà donnés ; on obtient la liste du nombre de points de charge par aménageur par commune. 
```sql
create or replace view VPaysLoire as
select * from i2a06b.VLA
UNION
select * from i2a06b.VMayenne
UNION
select * from i2a02a.VSarthe
UNION
select * from i2a02b.VMaine
UNION
select * from i2a04a.VVendee;
```

### Donner les droits
On donne les droits de sélectionner la vue globale aux autres utilisateurs afin de consulter les données. 
```sql
grant select on VPaysLoire to i2a02a;
grant select on VPaysLoire to i2a04a;
grant select on VPaysLoire to i2a02b;
```
