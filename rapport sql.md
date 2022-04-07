# Rapport mini projet SQL

## Table Recharge
Recharge correspond à une table temporaire dans laquelle les erreurs sont corrigées. Au lieu de créer des nouvelles tables à partir de basetd rechargeelectrique on le fait à partir de Recharge pour ne pas copier les erreurs. 

### Création de la table
```sql
create table recharge as select * from basetd.rechargeelectrique; 
```
### Correction des erreurs

```sql
/* deux communes ont le code insee 85234 et deux autres le code insee 85223 */
select commune, adresse, insee from recharge where insee in ('85234.0', '85223.0');

select * from recharge where département = 'La Réunion';

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
```sql
grant select on recharge to i2a07a;
grant select on recharge to i2a02b;
grant select on recharge to i2a02a;
grant select on recharge to i2a04a;
```
## Table Commune

### Création de la table
```sql
create table commune as
select distinct insee, commune as nomcommune, code_dep as numdep from recharge;
```
### Contraintes d'intégrité en local
```sql
alter table commune add constraint Commune_PK primary key (insee);
```

### Contraintes d'intégrité distantes
```sql
alter table commune add constraint Commune_FK foreign key (numdep) references departement(numdep);
```
### Donner les droits

```sql
grant select on commune to i2a07a;
grant select on commune to i2a02b;
grant select on commune to i2a02a;
grant select on commune to i2a04a;

GRANT REFERENCES(insee) ON commune to i2a04a;
```

## Table Département

### Création de la table
```sql
create table departement as
select distinct code_dep as numdep, département as nomdep from i2a06b.recharge;
```

### Contraintes d'intégrité en local
```sql
alter table departement add constraint Departement_PK primary key (numdep);
```

### Donner les droits

```sql
grant select on departement to i2a07a;
grant select on departement to i2a02b;
grant select on departement to i2a06b;
grant select on departement to i2a04a;

GRANT REFERENCES(numdep) ON departement to i2a06b;
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

```sql
/* i2a06b */
grant select on commune to i2a02a with grant option;
grant select on commune to i2a04a with grant option;
grant select on commune to i2a02b with grant option;

/* i2a04a */
grant select on station to i2a02a with grant option;
grant select on station to i2a06b with grant option;
grant select on station to i2a02b with grant option;
grant select on VVendee to i2a06b with grant option;

/* i2a02a */
grant select on VSarthe to i2a06b with grant option;

/* i2a02b */
grant select on VMaine to i2a06b with grant option;
```

### Créer la dernière vue

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

```sql
grant select on VPaysLoire to i2a02a;
grant select on VPaysLoire to i2a04a;
grant select on VPaysLoire to i2a02b;
```