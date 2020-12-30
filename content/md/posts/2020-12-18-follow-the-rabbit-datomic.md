{:title "Follow the white rabbit in Datomic"
:layout :post
:date "2020-12-18"
:tags ["clojure" "datomic"]}

Après le [Meetup](https://github.com/Charlynux/meetup-crafters-datalog) autour de Datomic, j'ai eu envie d'explorer les capacités d'analyse autour des transactions.

# Une requête ratée

Je voulais connaître toutes les modifications apportées par une transaction. De prime abord, ça ressemble à une requête assez simple. Voici ce à quoi je suis arrivé.

``` clojure
(d/q '[:find ?attr ?value
       :where
       [_  ?a ?value ?tx]
       [?a :db/ident ?attr]]
     (d/db conn)
     interesting-tx)
```

Cette requête échoue pour la raison suivante `Insufficent bindings, will cause db scan`. Datomic s'appuie sur de nombreux [index](https://docs.datomic.com/on-prem/indexes.html) (par entité, par attribut, ...) mais aucun par transaction.

# Log API

Après quelques recherches, je suis tombé sur la [Log API](https://docs.datomic.com/on-prem/log.html). On y trouve une requête qui ressemble à ce que je recherche.

``` clojure
(d/q '[:find ?e
       :in $ ?log ?t1 ?t2
       :where [(tx-ids ?log ?t1 ?t2) [?tx ...]]
              [(tx-data ?log ?tx) [[?e]]]]
     (d/db conn) (d/log conn) #inst "2013-08-01" #inst "2013-08-02")
```

Cette requête cherche toutes les entités modifiées entre deux dates.

Mais lorsque j'essaie de l'utiliser, j'obtiens une erreur m'indiquant que la fonction `log` n'existe pas.

C'est parce que dans mon cas, le `d` représente `datomic.client.api` or dans l'exemple il représente `datomic.api`.

# Un peu d'architecture

C'est à ce moment que le titre de cet article s'explique. Pour comprendre cette nuance dans les namespaces, il va falloir commencer à regarder l'architecture de Datomic. Ci-dessous, vous trouverez une illustration tirée de la [documentation officielle](https://docs.datomic.com/on-prem/architecture.html).

![Datomic Architecture](https://docs.datomic.com/on-prem/clientarch_orig.png)

Nous allons nous concentrer sur ce qui nous intéresse aujourd'hui. La librairie que nous utilisons aujourd'hui se trouve en haut à droite sur le diagramme, c'est la partie "Client". Le code que nous voudrions utiliser se trouve à gauche, c'est le "Peer app process".

Si l'on résume la chose un peu grossièrement, la Peer API est plus proche de Datomic, puisque plus proche du serveur. Il est donc logique que cette API permette des manipulations plus sophistiquées. (Il faut savoir qu'initialement, il n'y avait pas de Client API dans Datomic. Tout se faisait au niveau du "Peer".)

# Datomic dev-local vs Datomic Starter

Pour rappel, lors du Meetup, je m'appuyais sur Datomic dev-local, qui est le moyen le plus simple de manipuler Datomic sans avoir à installer et lancer X process (cf Architecture). Malheureusement, cette solution ne fournit pas la Peer API (puisqu'il n'y a pas de Peer.

Il existe deux autres méthodes pour faire tourner (gratuitement) Datomic sur votre machine : Datomic Free et Datomic Starter. Je n'ai pas encore creusé la première, mais la seconde est un Datomic complet, sa seule limite étant que les mises à jour sont limitées à un an.

Pour l'expérience, je me suis lancé dans Datomic Starter. Après quelques modifications dans le code du Meetup, j'ai pu faire tourner ma requête "Log".

# Différences d'API

Pour faire fonctionner le code avec Datomic Starter, j'ai dû faire de grosses modifications dans le code. En effet, si les APIs sont très semblables, elles divergent sur des détails qui nous obligent à une ré-écriture. Par exemple, la fonction `d/transact`, qui nous sert à insérer les données, prend une map dans l'API Client et une list dans l'API Peer.

J'ai donc nettoyé le code pour séparer les manipulations simples (celles du Meetup) et la requête "Log" en utilisant d'une part l'API Client et d'autre part l'API Peer. Les deux pointant vers la même base Datomic. Notez que ceci doit être plus proche de l'usage réel de Datomic.

Tout le code est disponible dans le [repository](https://github.com/Charlynux/meetup-crafters-datalog) du Meetup.

# To Be Continued

Afin d'approfondir ma compréhension de Datomic, il faudrait que je me plonge dans les talks de Rich Hickey, mais ils sont d'un certain niveau.

Pour l'instant, j'efforce de suivre la session "Day of Datomic" animée par Stuart Halloway ([lien](https://www.youtube.com/watch?v=yWdfhQ4_Yfw&list=PLZdCLR02grLoMy4TXE4DZYIuxs3Q9uc4i&ab_channel=ClojureTV)).
