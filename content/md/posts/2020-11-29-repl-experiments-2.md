{:title "REPL Experiments #2"
:layout :post
:date "2020-11-29"
:tags ["clojure" "repl" "java" "jsf"]}

Après avoir exploré les possiblités du REPL sur un programme Clojure en cours d'exécution ([ici](/posts/2020-11-28-repl-experiments-first.md)) Nous allons voir comment "brancher" Clojure sur une application Java.

Une des inspirations pour cette expérience est le tweet suivant.

<blockquote class="twitter-tweet" data-conversation="none"><p lang="en" dir="ltr">💯 I have a talk proposal floating around about saving literally person-years of labor by inserting an NREPL server into a terrible legacy Java system so I could live code it.</p>&mdash; ⸘Jack Rusher‽ (@jackrusher) <a href="https://twitter.com/jackrusher/status/1263877097933754370?ref_src=twsrc%5Etfw">May 22, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## L'application

Je n'avais de "terrible legacy Java system" sous la main. Je me suis donc rabattu sur une petite application en JSF (technologie que je ne connais pas) écrite par un enseignant pour ses élèves de Licence 2.

Elle gère des étudiants (les utilisateurs) et leurs notes.

## Honnêteté intellectuelle

Ce post pourrait laisser croire que cet exercice a été facile. Il n'en est rien. J'ai suivi plusieurs fausses pistes. Puis j'ai réussi à lancer un serveur NREPL dans l'application. Enfin après quelques simplications, je suis parvenu à lancer un Socket REPL (sans autre dépendance que Clojure) et à choisir (via un profile Maven) quand cette configuration est inclus ou non.

## Mode opératoire

### Clojure

Avant toute chose, il nous faut ajouter la dépendance Clojure. S'agissant d'un projet Maven, on ajoute les lignes dans le pom.xml.

```maven-pom
<dependency>
    <groupId>org.clojure</groupId>
    <artifactId>clojure</artifactId>
    <version>1.10.1</version>
</dependency>
```

### Lancer le REPL

Je ne suis pas parvenu à lancer le REPL via les arguments. Je me suis donc rabattu sur une solution en Java.

Cette solution est détournée d'un snippet en [Scala](https://blog.michielborkent.nl/clojure-from-scala.html) et ressemble beaucoup à la méthode pour lancer un [NREPL](https://nrepl.org/nrepl/usage/server.html#embedding-in-a-java-application).

``` java
IFn require = Clojure.var("clojure.core", "require");
require.invoke(Clojure.read("clojure.core.server"));
IFn start = Clojure.var("clojure.core.server", "start-server");
start.invoke(Clojure.read("{:port 5555 :accept clojure.core.server/repl :name repl :server-daemon false}"));
System.out.println("Clojure REPL server started on port 5555");
```

### Intégrer du code au démarrage de l'application

Le code ci-dessus doit être exécuté au démarrage de l'application.

La solution, qui m'a semblé la plus appropriée, a été de m'appuyer sur un "ManagedBean". Ce sont des objets instanciés par JSF, en plaçant le code dans le constructeur, on lancera le REPL au démarrage de l'application.

``` java
import clojure.java.api.Clojure;
import clojure.lang.IFn;
import javax.faces.bean.ManagedBean;
import javax.faces.bean.ApplicationScoped;

@ManagedBean(name="clojure-repl", eager=true)
@ApplicationScoped
public class ClojureRepl {
    public ClojureRepl() {
         // Code ci-dessus
    }
}
```

`@ApplicationScoped` indique que l'objet est instantié pour la durée de vie de l'application.

`eager=true` assure que l'objet est instantié sans attendre d'être demandé par quelqu'un.

### Conditionner le démarrage du REPL

La technique la plus simple que j'ai trouvée consiste à ajoute à la demande un dossier contenant ma ClojureRepl.java à la compilation.

``` maven-pom
<profiles>
    <profile>
        <id>clojure-repl</id>
        <dependencies>
            <dependency>
                <groupId>org.clojure</groupId>
                <artifactId>clojure</artifactId>
                <version>1.10.1</version>
            </dependency>
        </dependencies>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.codehaus.mojo</groupId>
                    <artifactId>build-helper-maven-plugin</artifactId>
                    <version>3.2.0</version>
                    <executions>
                        <execution>
                            <phase>generate-sources</phase>
                            <goals>
                                <goal>add-source</goal>
                            </goals>
                            <configuration>
                                <sources>
                                    <source>src/local/java</source>
                                </sources>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

Je peux donc démarrer le projet avec la commande suivante :

``` cmd
mvn -P clojure-repl jetty:run
```

## Utilisons ce REPL

Je ne suis pas parvenu au niveau de Jash Rusher et à "live coder" directement l'application.

La classe UtilisateurJPA est un [Singleton](https://en.wikipedia.org/wiki/Singleton_pattern), qui stocke les utilisateurs dans une liste.

L'utilisateur possède une méthode `getMoyenne` qui calcule et retourne sa moyenne.

``` clojure
(import '[the.package.jpa UtilisateurJPA])
(import '[the.package.modele Utilisateur])

;; Un utilisateur qui a toujours 20 de moyenne.
(def superuser (proxy [Utilisateur] []
                 (getMoyenne [] (double 20))))

(def me (doto superuser
          (.setLogin "charles")
          (.setMotDePasse "test")))

;; Ajout de cet utilisateur
(.create (UtilisateurJPA.) superuser)
```

Après cet ajout, je peux m'authentifier avec charles/test et on m'affiche bien une moyenne de 20.

## Conclusion

Je n'ai pas pu aller aussi loin que j'aurais voulu et "live codé" l'application. Mais j'ai quand même pu manipuler l'application déployée.

J'ajouterais que maintenant j'ai une meilleure vision de ce qu'implique d'ajouter un REPL à une application Java.
