{:title "REPL Experiments #1"
:layout :post
:date "2020-11-28"
:tags ["clojure" "repl"]}

Lorsqu'on entre dans le monde Clojure, on entend parler du "REPL" et des nombreux avantages qu'il présente.

Je ne prétendrais pas vous expliquer en quoi le REPL Clojure est différent des REPL qu'on peut trouver dans d'autres langages.

Personnellement, je me contente d'utiliser le REPL en local. Dans mon éditeur, je peux évaluer des morceaux de code un à un, me permettant ainsi de construire mes programmes par petites étapes.

Mais certains vont beaucoup plus loin dans leur usage. Ils s'en servent pour débugger leur code en production ou encore pour travailler en Clojure sur d'autres langages de la JVM (Java, Scala, ...).

J'entame ici une série d'expérimentations autour de ce sujet. L'objectif étant de me permettre de mieux appréhender ce qu'il est possible de faire avec le REPL.

## Notre cas

Pour une première expérience, j'ai essayé de faire simple.

Récemment, j'ai publié un JAR exécutant du code Clojure. Le JAR peut être trouvé [ici](https://github.com/Iteracode/nightcode-2020-api-examples).

Ce programme tourne indéfiniment. Toutes les minutes, il exécute des tests sur une url et affiche les résultats dans la console. Nous allons essayer de nous brancher à ce programme pour l'explorer voire le modifier.

Je précise que les actions suivantes n'ont nécessité aucune modification dans le code. Le JAR est celui publié il y a 2 mois.

## Comment procéder ?

La [documentation officielle](https://clojure.org/reference/repl_and_main) nous indique qu'il suffit d'ajouter un paramètre à l'exécution du programme.

```cmd
java -Dclojure.server.repl="{:port 5555 :accept clojure.core.server/repl}" -jar nightcode-demo.jar
```

## Ca a fonctionné ?

```cmd
telnet localhost 5555
```

Nous arrivons dans un REPL Clojure. Le paramètre est donc bien pris en compte.

Pour la suite, nous nous connecterons au REPL depuis un éditeur de texte (Emacs + inf-clojure, dans mon cas) plutôt que de taper directement dans le terminal.

## Exploration

Idéalement, lorsque vous vous connectez à un REPL depuis votre éditeur, vous le faites avec le code source associé au REPL.

Mais ce n'est pas obligatoire, voyons ce que l'on peut faire sans.

### Namespaces

On peut lister les namespaces du projet.

```clj
(all-ns)
```

On peut se limiter à ceux que nous avons implémenté.

```clj
(defn source? [a-ns]
    (let [ns-name (str a-ns)]
        (or (.startsWith ns-name "iteracode")
            (.startsWith ns-name "nightcode")
            (.startsWith ns-name "tavern"))))

(def my-namespaces
    (->>
        (all-ns)
        (filter source?)))
```

A partir de cette liste, on peut afficher toutes les fonctions.

```clj
(mapcat #(vals (ns-publics %)) my-namespaces)
```

### Analyse d'une fonction

Pour l'exemple, j'ai choisi la fonction `iteracode.examiner.report/report-results-console`.

```clj
(source iteracode.examiner.report/report-results-console)
```

J'avais exclu le code source du JAR, l'appel précédent retourne donc "Source not found".

Cette fonction n'est pas documentée, mais on peut au moins obtenir sa signature.

```clj
(doc iteracode.examiner.report/report-results-console)
```

```
-------------------------
iteracode.examiner.report/report-results-console
([results])
```

### Modification d'une fonction

Dans le code suivant, on change le comportement de la fonction.

```clj
(in-ns 'iteracode.examiner.report)

(def report-results-console-original report-results-console)

(defn report-results-console [results]
    (println "COUCOU")
    (report-results-console-original results))
```

A l'exécution suivante des tests, notre modification a été prise en compte.

```
Tests running...
COUCOU
FAIL - Simple hello, world test
```

## Conclusion

Sans disposer du code source, nous avons pu nous connecter au programme pendant son exécution et même modifier son comportement.

Dans le monde réel, le projet tournerait sur un serveur distant. Dans une prochaine expérimentation, nous essaierons d'exposer le REPL via un tunnel SSH.
