
# Utiliser SQLite sur iOS (partie 1)

SQLite est un système de gestion de bases de données.
SQLite est intégré dans iOS et utilisable directement dans les applications en swift.

Dans cette première partie nous allons voir les bases pour gérer une base de données dans une app iOS.
Dans une seconde partie nous verrons comment améliorer l'usage.


## Importer le module

La première chose à faire c'est d'importer le module :

```swift
import SQLite3
```


## Ouvrir une connexion

```swift
let path = "path/to/my/db.sqlite"
var connectionPointer: OpaquePointer!

if sqlite3_open(path, &connectionPointer) == SQLITE_OK {
    print("Connecté")
}
```

Il faut créer un objet du type `OpaquePointer` et le passer à la fonction `sqlite3_open()`.
`sqlite3_open()` renvoit la valeur de la constante `SQLITE_OK` si la connexion réussit.
`connectionPointer` représente alors la connexion à la base de données.


## Récupérer les erreurs

```swift
let err = sqlite3_errmsg(connectionPointer)
let error = String(cString: err)
```

La fonction `sqlite3_errmsg()` renvoit le dernier message d'erreur généré par SQLite.
La fonction prend un paramètre un pointeur vers une connexion (initialisé avec `sqlite3_open()`).


## Fermer la connexion

```swift
sqlite3_close(connectionPointer)
```

La fonction `sqlite3_close()` ferme une connexion.
La fonction prend un paramètre un pointeur vers une connexion (initialisé avec `sqlite3_open()`).
Après l'appel à `sqlite3_close()` `connectionPointer` ne désigne plus une connexion valide.


## Préparer une requête

```swift
let query = "SELECT * FROM table"
var statementPointer: OpaquePointer!

if sqlite3_prepare_v2(connectionPointer, query, -1, &statementPointer, nil) == SQLITE_OK {
    print("Requête préparée")
}
```

La fonction `sqlite3_prepare_v2()` prend en paramètre un pointeur vers une connexion ouverte,
une requête SQL sous forme de `String`, et un pointeur.
La fonction renvoit la valeur de la constante `SQLITE_OK` si tout est bon.
Après l'appel à `sqlite3_prepare_v2()` le pointeur représente la requête préparée.
On appelle une requête préparée un "statement".


## Finaliser un statement

```swift
sqlite3_finalize(statementPointer)
```

Il est nécessaire de finaliser les statements après usage.
Après appel de `sqlite3_finalize()` `statementPointer` ne désigne plus un statement valide.


## Exécuter un statement

Pour exécuter un statement, utiliser la fonction `sqlite3_step()` :

```swift
sqlite3_step(statementPointer)
```

Si la requête retourne une ligne de données, `sqlite3_step()` renvoit la valeur de la constante `SQLITE_ROW`.
Sinon, la fonction renvoit la valeur de la constante `SQLITE_DONE`.

Exemple d'une requête qui ne renvoit pas de données :

```swift
if sqlite3_step(statementPointer) == SQLITE_DONE {
    print("Requête exécutée, aucune ligne retournée")
}
```

Exemple d'une requête qui renvoit des données :

```swift
while true {
            
    let stepResult = sqlite3_step(statementPointer)
    
    guard stepResult == SQLITE_ROW || stepResult == SQLITE_DONE else {

        break
    }

    if stepResult == SQLITE_ROW {
        
        // lire les données
        
    } else {
        
        // toutes les lignes ont été lues

        break
    }
}
```


## Lire les résultats d'une requête

```swift
let index: Int32 = 0
let value = sqlite3_column_text(statementPointer, index)
let text = String(cString: value)
```

La fonction `sqlite3_column_text()` permet de lire la valeur de la ligne courante
dans la colonne indiquée par le second paramètre.

`sqlite3_column_int()` permet de lire un nombre.

Attention, ici le premier index est 0.


## Injecter des paramètres dans la requête

Si la requête contient des paramètres, par exemple :

```
INSERT INTO table(col1, col2) VALUES(?,?)
```

il faut définir les paramètres avant d'exécuter la requête avec `sqlite3_bind_text()`, `sqlite3_bind_int()`, ou `sqlite3_bind_null()` :

```swift
let str = NSString(string: "valeur de la colonne 1").utf8String
let index: Int32 = 1
sqlite3_bind_text(statementPointer, index, str, -1, nil)
```

Attention, ici le premier index est 1 et non pas 0.


## Utiliser des paramètres nommés

Si on souhaite utiliser des paramètres nommés dans la requête, par exemple :

```
INSERT INTO table(col1, col2) VALUES(:col1, :col2)
```

il faut utiliser `sqlite3_bind_parameter_index()` pour récupérer l'index du paramètre à partir de son nom :

```swift
sqlite3_bind_parameter_index(statementPointer, ":col1")  // 1
sqlite3_bind_parameter_index(statementPointer, ":col2")  // 2
```



# Liens

- https://sqlite.org/c3ref/open.html
- http://www.sqlite.org/c3ref/errcode.html
- https://sqlite.org/c3ref/close.html
- https://sqlite.org/c3ref/prepare.html
- https://sqlite.org/c3ref/finalize.html
- https://sqlite.org/c3ref/step.html
- https://sqlite.org/c3ref/column_blob.html
- https://sqlite.org/c3ref/bind_blob.html

- https://www.raywenderlich.com/385-sqlite-with-swift-tutorial-getting-started
- https://github.com/stephencelis/SQLite.swift/

