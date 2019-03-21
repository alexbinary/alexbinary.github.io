
# Utiliser SQLite sur iOS (partie 2)

Dans la première partie nous avons vu comment utiliser SQLite sur iOS.
Nous avons utilisé directement l'API C fournie dans iOS.
Cette API pose plusieurs problèmes :

- connexions et statements sont représentés par un même type d'objet (pointeur)
- un même pointeur peut correspondre à un objet dans différents états (connexion ouverte ou fermée, statement préparé ou non)

A cause de ces limitations certaines erreurs de code ne peuvent pas être détectées par le compilateur.

Nous allons ici proposer une solution à ces problème.

Pour résoudre le premier point, créons une classe pour une connexion et une classe pour un statement.
Chacune possède en interne un pointeur qui correspond à l'objet.

```swift
class Connection {

    var pointer: OpaquePointer!
}

class Statement {

    var pointer: OpaquePointer!
}
```

Par rapport au deuxième point, nous décidons qu'une instance de `Connection` représente nécessairement une connexion ouverte.
Une connexion fermée n'a pas beaucoup d'intérêt de toute façon.

De même nous décidons qu'une instance de `Statement` correspond nécessairement à un statement préparé.
Un statement non préparé n'a pas beaucoup d'intérêt non plus.

Dans `Connection`, on ouvre la connexion dans l'`init`, et on ferme la connexion dans le `deinit`.
Dans `Statement` on prépare le statement dans l'`init`, et on libère le pointeur dans le `deinit`.

Ainsi on sait toujours dans quel état se trouve le pointeur.

Pour préparer un statement nous avons besoin de la connexion.

```swift
class Connection {

    var pointer: OpaquePointer!

    init(path: String) {

        sqlite3_open(path, &pointer)
    }

    deinit {

        sqlite3_close(pointer)
    }
}

class Statement {

    var pointer: OpaquePointer!

    init(query: String, connection: Connection) {

        sqlite3_prepare_v2(connection.pointer, query, -1, &pointer, nil)
    }

    deinit {
        
        sqlite3_finalize(pointer)
    }
}
```

Voilà déjà une bonne base.

Sur la connexion, on peut rajouter du code pour récupérer facilement le dernier message d'erreur :

```swift
class Connection {

    ...

    var errorMessage: String? {
        
        if let error = sqlite3_errmsg(pointer) {
            
            return String(cString: error)
            
        } else {
            
            return nil
        }
    }
}
```

Sur le statement, on peut ajouter une méthode pour exécuter le statement :

```swift
class Statement {

    ...

    func run() {
        
        sqlite3_step(pointer)
    }
}
```