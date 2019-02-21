
# Utiliser SQLite sur iOS (partie 3)

Dans la partie 2 nous avons vu comment encapsuler l'API C de SQLite afin de profiter de la détection d'erreur à la compilation.

Nous allons voir ici comment allez plus loin.

Dans la solution précédente, nous devons écrire les requêtes SQL à la main.
Le compilateur n'a aucun moyen de vérifier qu'une requête est correcte.

Nous allons proposer ici une solution.

L'idée est de créer des objets qui vont générer les requêtes pour nous, réduisant ainsi les risques d'erreur.
Ces objets vont s'appuyer sur d'autres objets qui décrivent la structure de la base de données.

Commençons par définir un enum pour les types de colonnes :

```swift
enum ColumnType {
    
    case bool
    
    case char(size: Int)

    ...
}

```

Créons ensuite un objet qui représente une colonne :

```swift
struct Column {
    
    let name: String
    
    let type: ColumnType
    
    let nullable: Bool
}
```

Finalement nous pouvons créer notre table :

```swift
struct Table {
   
    var name: String
    
    var columns: [Column]
}

```


Nous pouvons maintenant passer aux objets qui vont créer les requêtes.

Commençons par créer un protocol qui représente un objet qui peut s'exprimer avec du code SQL.
Le protocol fournit une propriété qui permet de récupérer la représentation textuelle de la requête :

```swift
protocol SQLRepresentable {

    var sqlString: String { get }
}
````

On peut d'ores et déjà ajouter la conformance sur nos objets `ColumnType` et `Column` :

```swift
extension ColumnType: SQLRepresentable {
    
    var sqlString: String {
        
        switch (self) {
            
        case .bool:
            
            return "BOOL"
            
        case .char(let size):
            
            return "CHAR(\(size))"

        ...
        }
    }
}

extension Column: SQLRepresentable {
    
    var sqlString: String {
        
        return [
            
            name,
            type.sqlString,
            nullable ? "NULL" : "NOT NULL",
            
        ].joined(separator: " ")
    }
}
```


Créons maintenant un objet qui représente une requête complète.
Cet objet se conforme au protocol précédent mais n'apporte rien, il est là juste pour permettre de typer les requêtes (utile dans la suite) :

```swift
protocol SQLQuery: SQLRepresentable {

}
````

Nous pouvons maintenant créer un objet qui représente une requête de type select.
L'objet prend en paramètre une table et produit une requête qui sélectionne toutes les colonnes :

```swift
struct SelectQuery {

    var table: Table

    var sqlString: String {

        return "SELECT * FROM \(table.name);"
    }
}
```

Créons maintenant une requête d'insertion.
La requête produite comprendra des paramètres pour les valeurs à insérer :

```swift
struct InsertQuery {

    var table: Table

    var sqlString: String {

        return [
            "INSERT INTO \(table.name) (",
            table.columns.map { $0.name } .joined(separator: ", "),
            ") VALUES (",
            table.columns.map { _ in "?" } .joined(separator: ", "),
            ");"
        ]
    }
}
```

Sur le même principe on peut créer une requête qui crée une table :

```swift
struct InsertQuery {

    var table: Table

    var sqlString {

        return [
            "CREATE TABLE \(table.name) (",
            table.columns.map { $0.sqlString } .joined(separator: ", "),
            ");"
        ]
    }
}
```

Finalement, nous pouvons modifier notre objet `Statement` pour prendre un objet du type `SQLQuery` :

```swift
class Statement {

    ...

    init(query: SQLQuery, connection: Connection) {

        sqlite3_prepare_v2(connection.pointer, query.sqlString, -1, &pointer, nil)
    }
}
```

Avec ça, impossible d'écrire une requête SQL invalide !