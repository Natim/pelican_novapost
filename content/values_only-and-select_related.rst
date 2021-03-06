===============================
Django, l'ORM et l'optimisation
===============================

:date: 2015-01-28 13:55
:tags: python, django, ORM, SQL
:category: Astuces
:author: Yohann Gabory


Comme vous le savez sans doute, les objets de type QuerySet sont
lazy. C'est à dire qu'ils ne sont évalués qu'au tout dernier moment.
En fait, ils peuvent même ne pas être évalués du tout. Ou au contraire
être évalués à de nombreuses reprises.

Evidemment, pour de meilleures performances, on va chercher à les
évaluer le plus tard et le moins possible.

Je vais essayer de montrer, au travers de quelques exemples qu'en n'y
prenant pas garde, le plus tard possible signifie souvent aussi de
nombreuses fois.

Je proposerai enfin une façon de se prémunir contre cet état de fait
et de mieux maîtriser où et quand un appel à la DB est effectué.

Première solution: Ne pas évaluer les querysets au moment du "render"

Dans cette première solution, afin de ne pas générer de requête
inutile, on va passer directement au template une queryset non
évaluée :

.. code-block:: python

    >> context = {"objects": Store.objects.all()}

Dans le template on peut ensuite écrire:

.. code-block:: django::

   {% for store in objects %}

    {{store.name}} {{store.open_time}}

   {% endfor %}

Jusqu'ici tout va bien.

Quelques itérations plus tard, on décide d'ajouter le nom de la "location"
du "store":

.. code-block:: django::

    {% for store in objects %}

    {{store.name}} {{store.open_time}} {{store.location.name}}

    {% endfor %}

Et là tout se corse. En effet, comme location est une autre table,
django va devoir faire une jointure et une requête supplémentaire pour
chacun des auteurs. Les performances de la page sont dégradées.

Heureusement, une parade existe:

Select Related
==============

Au moment de renvoyer la queryset, on peut utiliser select_related
pour récupérer la table Location:

.. code-block:: python

    >> context = {"objects":
    Store.objects.all().select_related("location")}

cette fois on revient à une seule requête plus gourmande cependant que
la première.

.. code-block:: sql

    SELECT "library_store"."id", "library_store"."name",
           "library_store"."location_id", "library_store"."close_time",
           "library_store"."open_time", "library_store"."open_date",
           "library_location"."id", "library_location"."name"
    FROM "library_store" INNER JOIN "library_location"
    ON ( "library_store"."location_id" = "library_location"."id"

sans le select_related nous avions:

.. code-block:: sql

    SELECT "library_store"."id", "library_store"."name",
           "library_store"."location_id", "library_store"."close_time",
           "library_store"."open_time", "library_store"."open_date"
           FROM "library_store"'

Vous ne trouvez pas qu'il y a beaucoup de choses qui ne nous servent
pas ? Ne pourrions-nous pas faire mieux ?

Values
======

l'utilisation de values va vous permettre de retourner au template une
liste de dictionnaires avec uniquement ce dont vous avez besoin:

dans notre dernier exemple, on a besoin de:

- ``{{store.name}}``
- ``{{store.open_time}}``
- ``{{store.location.name}}``

On peut donc écrire:

.. code-block:: python

    >> context = {"objects":
     Store.objects.all().values("name", "open_time", "location__name")}

la variable du template deviendra ``{{store.location__name}}``
et tout ira bien.

Quel gain en performance entre values et select_related ?

.. code-block:: python

    import logging
    l = logging.getLogger('django.db.backends')
    l.setLevel(logging.DEBUG)
    l.addHandler(logging.StreamHandler())
    Store.objects.all().select_related("location")
    (0.007) SELECT "library_store"."id",<snip>

    >>> Store.objects.all().values("name", "open_time", "location__name")
    (0.001) SELECT "library_store"."name"

Tests réalisés sur un postgresql avec 10.000 objets.

Values gagne donc haut la main, avec un rapport de 1/7 (plus on a de
champs et d'enregistrements, plus le rapport augmente.)

Seulement voilà, il y a quand même un mais.

Vous utilisez Django pour les formidables méthodes que vous avez
écrites amoureusement. Par exemple, vous avez sur le modèle Store une
méthode qui calcule le nombre d'heures ouvrées (la différence entre
open_time et closed_time)

Avec la première solution, pas de problème:

``{{store.open_hour}}``

en revanche, votre dictionnaire ne connaît pas la méthode "open_hour".

Première solution (la plus performante) SQL ne vous fait pas peur:

.. code-block:: python

    Store.objects.all().extra(
        {"open_hour": "close_time - open_time"}
        ).values("open_time", "open_hour")

faire une soustraction entre deux entiers, PosgreSQL se débrouille pas
trop mal ;)

en termes de performances ça donne (0.001) et vous n'avez aucun
retraitement à faire en python.

Only
====

Seconde solution, utiliser only.

.. code-block:: python

    Store.objects.all().only("open_time")

l'avantage de cette seconde solution : vous avez un vrai objet python
et vous pouvez appeler vos méthodes préférées.

Seulement voilà, en vrai il va se passer quelque chose de vraiment pas
sympa:

.. code-block:: python

    >>> a = Store.objects.all().only("open_time")
    >>> a[0].open_hour()
    (0.001) SELECT "library_store"."id", "library_store"."open_time" FROM "library_store" LIMIT 1; args=()
    (0.001) SELECT "library_store"."id", "library_store"."close_time" FROM "library_store" WHERE "library_store"."id" = 1 ; args=(1,)


En utilisant only vous avez dit à votre ORM: "Je te jure que je n'ai
besoin que de open_time, rien d'autre, promis". Mais vous lui avez
menti. Quelques secondes plus tard vous appeliez close_time pour votre
méthode. Django ne sachant que faire est contraint de faire une
seconde requête en base de données réduisant vos efforts à néant.

Si en revanche vous demandez les bonnes informations dès le départ
vous allez avoir une bonne surprise:

>>> a = Store.objects.all().only("open_time", "close_time")
>>> a[0].open_hour()
(0.001) SELECT "library_store"."id", "library_store"."close_time", "library_store"."open_time" FROM "library_store" LIMIT 1; args=()

Cette fois, vous n'avez pas menti et l'ORM vous gratifie d'une seule
et unique requête.

Le principal souci que vous allez rencontrer au moment de choisir
l'une ou l'autre des solutions, c'est que **si vous ne savez pas ce que
l'on va faire de votre requête** une fois qu'elle va être envoyée au
template **vous ne pouvez pas optimiser**.

Que ce soit avec only ou avec values.

L'avantage de only, c'est que vous pouvez encore utiliser vos
méthodes. Si vous avez récupéré ce dont vous avez besoin, c'est
parfait.

Mais only ne vous dira jamais qu'il lui manque un attribut. Il ira
tout simplement le chercher et ce, à chaque fois que vous en aurez
besoin.

imaginez le désastre de :

>>> lst = Store.objects.only("close_time")
>>> for a in lst:
...     a.open_time

dans ce cas vous auriez mieux fait de faire une requête "normale"

l'avantage indéniable de values, c'est que rien n'est caché. Vous
accédez à un attribut qui n'existe pas ?

Django vous renvoie une KeyError, simple et facile à tracer.

En revanche vous perdez vos méthodes. Ça peut vraiment être très
pénible. Ceci dit, si vous savez que vous allez avoir besoin d'une
méthode dans le template, pourquoi ne pas l'ajouter à votre
dictionnaire ?

>>> lst = Store.objects.all().values("name", "open_time", "close_time", "location__name")
>>> for a in lst:
...     a["open_hour"] = a["close_time"] - a["open_time"]
(0.019) SELECT "library_store"."name", <snip>

la même chose avec only

>>> lst = Store.objects.only("close_time", "open_time")
>>> for a in lst:
...     a.open_hour = a.close_time - a.open_time
(0.009) <snip>

ici, only est 2 fois plus rapide.

Conclusion
==========

Tout ceci pour dire:

- utilisez values quand vous n'avez pas besoin des méthodes
- sauf si ces méthodes peuvent être executées en SQL

- utilisez only si vous avez besoin de certaines méthodes et que vous
  êtes certain de ne pas avoir besoin d'autres champs, explicitement
  ou dans l'une des méthodes que vous allez utiliser.

- utilisez des requêtes "classiques" quand vous ne maîtrisez pas ce
  qui va se passer

- dans ce cas utilisez select_related autant que possible si vous
  savez que vous allez avoir besoin d'autres tables.

- utilisez la DDT pour tracer vos requêtes.

    - chassez les doublons (plusieurs fois la même requête avec un
      paramètre qui change) il vous manque un select_related

    - chassez les requêtes avec un SELECT très volumineux, essayez
      only, vous verrez passer des requêtes supplémentaires, ajoutez
      les attributs manquants à votre only

    - utilisez values dès que vous le pouvez. Vous ne pourrez pas
      faire mieux en terme de performance.

Comme vous l'avez vu, si vous n'avez pas besoin des méthodes de votre
objet python, caster une liste de dictionnaires avec values peut être
une bonne idée. N'oubliez pas que values reste une queryset, vous avez
encore le droit de filtrer!

>>> Store.objects.all().values(
    "name", "open_time", "location__name").filter(location__pk=1).first()

est parfaitement valable!


Le coin du cochon farceur
=========================

Ce qui suit n'est pas à conseiller aux âmes sensibles. Il s'agit de
tenter d'avoir le meilleur des deux mondes: des dictionnaires avec les
fonctions du modèle:

**CECI EST UN JEU DE L'ESPRIT, IL NE FAUT PAS LE FAIRE!!!**

reprenons notre classe Store:

.. code-block:: python

    class Store(models.Model):
        name = models.CharField(max_length=250)
        location = models.ForeignKey(Location)
        close_time = models.PositiveIntegerField(max_length=2)
        open_time = models.PositiveIntegerField(max_length=2)
        open_date = models.DateField()

        def open_hour(self):
            return self.close_time - self.open_time


et coupons la en 2:

.. code-block:: python

    class StoreMixin(object):
        def open_hour(self):
            return self.close_time - self.open_time


    class Store(StoreMixin, models.Model):
        name = models.CharField(max_length=250)
        location = models.ForeignKey(Location)
        close_time = models.PositiveIntegerField(max_length=2)
        open_time = models.PositiveIntegerField(max_length=2)
        open_date = models.DateField()


ajoutons un peu de sucre:

.. code-block:: python

    class DictToObj(StoreMixin):
        def __init__(self, **kwargs):
            self.__dict__.update(kwargs)


    >>> stores = Store.objects.all().values(
        "name", "open_time", "closed_time", "location__name")

    >>> template_stores = [DictToObj(**store) for store in stores]
    >>> template_stores[0].open_hour()
    12

Vous avez retrouvé vos objets (et moi je vais allé me cacher
parce que ce n'est pas joli, joli quand même.)

Pour les lecteurs pressés
=========================

Ce que j'ai voulu démontrer:

1) non select_related n'est pas magique
2) only est dangereux (comme son copain defer)
3) values reste la meilleure solution si on maîtrise ce que l'on fait.
4) extra peut faire des trucs vraiment sexy.
