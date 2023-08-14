---
marp: true
theme: gaia
_class: lead
paginate: true
backgroundImage: url('./assets/images/hero-background.svg')
footer: 01-06-2023 - Jardin d'Acclimatation
---
# Restitution JavaDay 2023 ☕

---
# JavaDay

- Évènement annuel organisé par le Paris JUG (2ème édition cette année)
- Conférences centrées autour de l'écosystème Java


---
# Java 21

## Evolution du langage
---
### Pattern matching dans les switch
```java
static void print(Object o) {
  switch (o) {
    case String s:
      System.out.println("string : " + s);
      break;
    default:
      System.out.println("other type : " + o);
      break;
  }
```
---
```java
static String testGuardedPattern(Object o) {
  return
    switch (o) {
      case String s when s.length() < 10 -> "short string : " + s;
      case String s when s.matches("\\d+") -> "integer string : + s";
      case String s -> "other string " + s;
      case Double d -> "double : " + d;
      default -> "other : " + o;
    };
}
```
---
### Record pattern
```java
  record Formateur(String nom, String prenom) {}
  Object o = new Formateur("Durand", "Pierre");
  if (o instanceof Formateur f) { // Type pattern
      System.out.println("Formateur : " + f.nom() + " " + f.prenom());
  }
  if (o instanceof Formateur(var nom, var prenom)) { // Record pattern
      System.out.println("Formateur : " + nom + " " + prenom);
  }
```

---
## Concurrence / Multi-threading (projet Loom)
### Threads virtuels
* Géré par la JVM VS thread classique géré par le système
* Thread virtuel non lié à un thread système => plus léger et pas besoin de pool de thread
* Objectif : meilleures performances sur les applications avec beaucoup ce concurrence
---
* 2 manières de les créer
```java
var vthread = Thread.ofVirtual()
        .name("mon-thread-virtuel-", 0)
        .start(() -> {
            System.out.println(Thread.currentThread());
        });

try (ExecutorService e = Executors.newVirtualThreadPerTaskExecutor()) {
    e.submit(() -> System.out.println(Thread.currentThread()));
}
```
--- 
* Limitations
  - La priorité est obligatoirement Thread.NORM_PRIORITY
  - Les méthodes stop(), resume(), suspend() lèvent une
UnsupportedOperationException

---
### Structured concurrency
* API pour un modèle de programmation multi-thread
  - Ecriture du code dans un style synchrone avec une exécution en asynchrone
  - Classe principale : StructuredTaskScope - implémente AutoCloseable

---
```java
static Produit getProduit(List<Fournisseur> fournisseurs, String reference) throws InterruptedException, ExecutionException {
        Produit resultat;
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<Produit>()) {
            fournisseurs.forEach(f -> scope.fork(() -> f.getProduit(reference)));
            scope.join();
            resultat = scope.result();
        }
        return resultat;
    }

```
* Scoped Value
  - API pour partager les données immuables entre threads
  - A préférer par rapport à ThreadLocal
---
```java
final static ScopedValue<...> V = ScopedValue.newInstance();
// In some method
ScopedValue.where(V, <value>)
.run(() -> { ... V.get() ... call methods ... });
// In a method called
... V.get() ...
```

---
## API
### Vector API (projet Panama)

```java
void vectorComputation(float[] a, float[] b, float[] c) {
    int i = 0;
    int upperBound = SPECIES.loopBound(a.length);
    for (; i < upperBound; i += SPECIES.length()) {
        var va = FloatVector.fromArray(SPECIES, a, i);
        var vb = FloatVector.fromArray(SPECIES, b, i);
        var vc = va.mul(va).add(vb.mul(vb)).neg();
        vc.intoArray(c, i);
    }
    for (; i < a.length; i++) {
        c[i] = (a[i] * a[i] + b[i] * b[i]) * -1.0f;
    }
}
```
---
### Foreign Function & Memory API

API bas niveau pour appeler des méthodes natives et leur réserver de la mémoire

---
```java
private static int msgBox(String message, String titre) {
    try {
        System.loadLibrary("user32");
        Linker linker = Linker.nativeLinker();
        SymbolLookup loaderLookup = SymbolLookup.loaderLookup();
        var msgBoxFct = loaderLookup.find("MessageBoxA");
        var descriptor = FunctionDescriptor.of(JAVA_INT, ADDRESS, ADDRESS, ADDRESS, JAVA_INT);
        var methodHandle = linker.downcallHandle(msgBoxFct.get(), descriptor);
        try (Arena offHeap = Arena.ofConfined()) {
            MemorySegment cStrMsg = offHeap.allocateUtf8String(message);
            MemorySegment cStrTitre = offHeap.allocateUtf8String(titre);
            return (int) methodHandle.invoke(NULL, cStrMsg, cStrTitre, 0x00000024);
        }
    } catch (RuntimeException re) {
        throw re;
    } catch
    (Throwable t) {
        throw new RuntimeException(t);
    }
}
```
---
### Evolutions d'API
* Nouvelle propriété système -Djava.properties.date : contrôle du format de date pour écriture dans les fichiers .properties (Properties::store)
* Ajout de méthodes
  - nouvelles surcharges de of dans Locale
  - HashSet / HashMap : 
    * HashMap.newHashMap(int) / HashSet.newHashSet(int)
    * LinkedHashMap.newLinkedHashMap(int) / LinkedHashSet.newLinkedHashSet(int)

---
### Evolutions JVM
* UTF-8 devient le charset par défaut
* Renforcement de la sécurité
  - JARs signés avec SHA-1 désactivés
  - Augmentation de la taille de clés par défaut de KeyPairGenerator et KeyGenerator
* Méthode fializalize() / runFinalization() dépréciée => remplacer par try-with-resource ou API Cleaner

---
* Javadoc
  - Ajout de @snippet pour insérer du code dans la doc
  - Ajout de tags de mise en forme de la doc : @highlight, @replace, @link
* Serveur Web minimaliste
  - Uniquement GET et HEAD et protocole HTTP 1.1
  - Pas de HTTPS, port 8000 par défaut
```java
var server = SimpleFileServer.createFileServer(
      new InetSocketAddress(8000),
      Path.of("c:/tmp"),
      SimpleFileServer.OutputLevel.INFO);
server.createContext("/java", SimpleFileServer.createFileHandler(Path.of("c:/tmp")));
server.start();
```
---
# Jakarta EE 10

## Timeline

![auto](./assets/images/timeline_jakarta_1.png)
![auto](./assets/images/timeline_jakarta_2.png)


---
<!-- _footer: Jakarta EE 10 -->
## Components

* Jakarta Security
  - OpenID en standard
* Jakarta Persistence
  - UUID Type
* Jakarta RestFul WS
  - Multipart
* CDI Lite
  - Subset of CDI

![bg 80% right:60%](./assets/images/jakarta-ee-10.png)

<!-- 
Jakarta: Specifications, meaning that use these annotations as much as possible 

Core Profile: new minimal profile for microservices
-->
---
# GraalVM and Spring Boot Native


---
# CRaC vs GraalVM

- CRaC et GraalVM, même objectif : démarrage rapide
- Se passer de la "pré-chauffe" de la JVM :
    - JIT compilation
    - Analyse des annotations
    - Initialisation d'un contexte applicatif (Spring)


---
# CRaC vs GraalVM

- Checkpoint & restore : Faire une "snapshot" de la JVM après ce warm-up

- Démarrage en mode restore depuis le checkpoint : on passe cette phase de warm-up

- Permets de se passer des contraintes du natif (on travaille toujours avec une JVM)




---
# Supply Chain and SBOM

![bg 55%](./assets/images/sbom.webp)

<!-- cf. Blackduck, dependencyTrack, etc. -->
---
<!-- _footer: Supply Chain and SBOM -->
# Supply Chain and SBOM

![bg 80%](./assets/images/supply_chain.png)

---
<!-- _footer: Supply Chain and SBOM -->
# Supply Chain and SBOM

![bg 80%](./assets/images/supply_chain_digital.png)

<!-- Idée: faciliter la commercialisation, identification des defects, etc. -->
---
<!-- _footer: Supply Chain and SBOM -->
# PGP vs Sigstore

![bg 40%](./assets/images/pgp.png)

<!-- Echange de clé: chiant, contrainte: jamais perdre la clé privé -->
---
<!-- _footer: Supply Chain and SBOM -->
# PGP vs Sigstore

![bg 80%](./assets/images/sigstore.png)

<!-- Signature "sans clés"
Fulcio: certificat a durée de vie limitée: 10m
Rekor: stocke les certifs dans un transparency log

Encore des problématiques a résoudre: Qui croire? 
Et reste toujours un "probleme": checker les signatures! -->
---
