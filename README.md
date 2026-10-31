# Exercice : Design Uber API

## Partie A : Périmètre et deux faces du produit

**1. Pourquoi clarifier le périmètre avant de dessiner un schéma ?**  
Uber regroupe plusieurs produits (courses, Eats, Pool) qui n'ont pas les mêmes données ni les mêmes règles. Si on dessine sans avoir cadré on risque de tout mélanger.

**2. Périmètre choisi**  
Pour cet exercice je me limite uniquement à l'API des courses (core rides API).

**3. Actions des deux côtés**  
Le passager peut demander une course, consulter son statut et l'annuler. Le conducteur peut se mettre en ligne, voir les courses disponibles, accepter une course et la terminer.

**4. Pourquoi deux API distinctes ?**  
Parce que le passager et le conducteur ne font pas les mêmes actions. Par exemple, seul le conducteur peut accepter une course, pas le passager. Et en séparant les deux API, on peut modifier l'une sans casser l'autre.

## Partie B : L'entité Ride

**5. Champs de Ride**  
Le schéma complet dans `api.txt`. J'ai mis l'identifiant, le passager, le conducteur, le statut, le départ, la destination, le prix et les dates de chaque étape. Le conducteur est vide au début, car personne n'a encore accepté. Le prix est un entier en centimes pour éviter les erreurs d'arrondi.

**6. Référence ou objet imbriqué ?**  
Une simple référence : `passenger_id` et `driver_id` à la place de `passenger_info` et `driver_info`. Comme ça, les infos du passager et du conducteur sont stockées une seule fois, et la course les retrouve avec leur id. C'est plus léger, et si une info change (un numéro de téléphone par ex.), on la modifie à un seul endroit.

**7. Endpoints minimum**  
J'utilise POST quand on crée une course ou qu'on change son état, et GET quand on lit seulement.

| Côté       | Endpoint             | Méthode | Objectif                         |
| ---------- | -------------------- | ------- | -------------------------------- |
| Passager   | /rides               | POST    | Demander une course              |
| Conducteur | /rides/{id}/accept   | POST    | Accepter une course              |
| Passager   | /rides/{id}          | GET     | Consulter le statut d'une course |
| Conducteur | /rides/{id}/complete | POST    | Terminer la course               |

**8. Cycle de vie (statuts)**  
requested, accepted, in_progress, completed. J'ajoute `cancelled` pour une course annulée avant le départ (une course peut être annulée tant que le trajet n'a pas commencé, donc avant in_progress).

## Partie C : Uber Pool, côté passager

**9. Peut-on transformer passenger_info en liste ?**  
Non, car une Ride a un seul départ, une seule destination, un seul prix et un seul statut. Avec plusieurs passagers, chacun a son propre trajet, son propre prix et son propre état : un peut déjà être dans la voiture pendant que l'autre attend encore. Une seule ride ne peut pas porter tout ça

**10. Meilleure architecture**  
Je crée une entité PoolRide qui contient un tableau de Ride, une par passager. Et le PoolRide les regroupe dans la même voiture.

```
PoolRide
├── driver_id (un seul conducteur)
├── rides
│   ├── Ride (passager A)
│   └── Ride (passager B)
├── dropoff_order
├── max_capacity
└── available_seats
```

**11. Infos supplémentaires dans PoolRide**  
Un id, le conducteur (le même pour toutes les rides, à la place de `driver_info`), l'ordre de dépose, la capacité du véhicule et le nombre de places vides.

**12. Pourquoi réutiliser Ride ?**  
Parce que la logique d'une course (statuts, prix, dates, endpoints) existe déjà et marche. Inutile de la réécrire et de maintenir deux versions.

## Partie D : Uber Pool, côté conducteur

**13. Nouveau statut conducteur**  
Je propose `on_ride_available` : le conducteur est déjà en course, mais il lui reste des places, donc il peut prendre un passager de plus. Les statuts deviennent offline, available, on_ride (voiture pleine) et on_ride_available.

**14. Comment il découvre de nouvelles courses ?**  
Il consulte les courses disponibles avec GET /rides/available et accepte avec le même endpoint qu'une course simple (POST /rides/{id}/accept). La différence, c'est que le serveur filtre selon son statut : en on_ride_available, il voit seulement les courses proches de son trajet, sans gros détour. Quand il accepte, la ride est rattachée à son PoolRide en cours.

**15. Quand arrêter d'accepter ?**  
Quand la voiture est pleine (available_seats = 0), ou quand le détour imposé aux passagers déjà dans la voiture devient trop long, par exemple plus de 10 minutes. Son statut repasse alors à on_ride.
