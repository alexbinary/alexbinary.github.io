# iOS: Chargement d'images avec cache et usage dans un table view - partie 2


Dans un précédent article, nous avons vu comment télécharger des images avec un
système de cache pour un usage dans un table view.

Aujourd'hui nous allons améliorer le système afin d'offrir une meilleure
expérience aux utilisateurs.


# Les problèmes de l'implémentation actuelle

Pour rappel, voici l'implémentation de notre fournisseur d'images et un exemple
d'usage dans une cellule de table view :


```swift
class ImageLoader {

    static let shared = ImageLoader()

    let cache = NSCache<NSURL, UIImage>()

    func image(for url: URL, completionHandler: @escaping (UIImage) -> Void) {

        if let image = cache.object(forKey: url as NSURL) {

            completionHandler(image)

        } else {

            downloadImage(from: url, completionHandler: completionHandler)
        }
    }

    func donwloadImage(from url: URL, completionHandler: (UIImage) -> Void) {

        URLSession.shared.dataTask(with: url) { data, response, error in
            
            let image = UIImage(data: data!)!
            
            self.cache.setObject(image, forKey: url as NSURL)

            completionHandler(image)

        } .resume()
    }
}
```

```swift
class MyCell: UITableViewCell {

    var imageURL: URL {
        didSet {
            ImageLoader.shared.image(for: imageURL) { image in 
                DispatchQueue.main.async {
                    self.mainImageView.image = image
                }
            }
        }
    }
}
```


Vu que les cellules d'un table view sont détruites et réutilisées dès qu'elles
sortent de l'écran, il est possible qu'on définisse une nouvelle valeur pour la
propriété `imageURL` alors même que l'image précédente est encore en train de 
charger. On peut alors voir apparaitre l'image précédente un bref instant avant
qu'elle soit remplacée par la nouvelle image. Du point de vue de l'utilisateur,
il va voir apparaitre une image qui ne correspond pas à la cellule dans laquelle
elle apparait. Dans de rares cas, il est même possible que la requête de chargement
de la seconde image finisse avant celle de la première image, auquel cas on verra
d'abbord apparaitre la seconde image, qui est l'image voulue, avant qu'elle soit
remplacée par la première image, qui n'est pas l'image voulue, et la cellule
affichera alors la mauvaise image tant qu'on ne met pas à jour la propriété
`imageURL`.

Pour éviter d'avoir la mauvaise image quand la cellule est réutilisée, il faut
annuler le chargement de l'image s'il est encore en cours lorsque la valeur de
la propriété `imageURL` est mise à jour.


# Annuler une requête de chargement d'image

Implémentons donc la possibilité d'annuler une requête de chargement d'image dans
notre fournisseur d'image. 

Pour annuler une requête, nous devons pouvoir l'identifier. La méthode qui initie
le chargement d'image va donc nous renvoyer un identifiant de requête que nous
allons pouvoir utiliser pour annuler la requête par la suite.

Lorsqu'une requête est annulée, notre fournisseur d'image n'appelera pas la closure
avec l'image téléchargée. Comme dans le cas d'un table view l'image a de fortes chances
de devoir être affichée à nouveau par la suite, nous allons tout de même aller au
bout du téléchargement et mettre l'image en cache, même si elle n'est pas utilisée
immédiatement. Si l'image est demandée à nouveau, elle sera servie du cache.
On évite ainsi d'annuler la requête en plein milieu et ainsi de télécharger
deux fois une partie de l'image.

Une optimisation plus poussée consiste à mettre
en pause le téléchargement lorsqu'on annule la requête, et à le reprendre lorsque
l'image est demandée plus tard. Ainsi on ne télécharge rien deux fois, et on ne 
télécharge rien inutilement. La mise en place de ce système est plus complexe et
nous n'allons pas le faire ici.

Nous apportons les modifications suivantes :

- au lieu de passer directement la closure à la méthode qui télécharge l'image,
nous l'ajoutons dans une liste
- au lieu d'appeler directement la closure quand l'image est téléchargée, nous
récupérons la closure depuis la liste

Pour annuler une requête, il nous suffit alors de supprimer la closure de la liste.


Commençons par générer un identifiant de requête. Nous utilisons pour cela un
UUID, qui permet de générer un identifiant unique :

```swift
class ImageLoader {

    func image(for url: URL, completionHandler: @escaping (UIImage) -> Void) -> UUID {

        let requestId = UUID()

        if let image = cache.object(forKey: url as NSURL) {

            completionHandler(image)

        } else {

            downloadImage(from: url, completionHandler: completionHandler)
        }

        return requestId
    }

    ...

}
```

Ajoutons ensuite la liste des closures. Nous utilisons un dictionaire
qui nous permet d'associer chaque closure à l'identifiant de la requête correspondante.

La méthode `image(for:completionHandler:)` ajoute la closure à la liste.
La méthode `donwloadImage(from:completionHandler:)` devient `donwloadImage(from:requestId:)`.
Elle ne prend plus directement la closure en argument, mais la récupère d'après l'identifiant de requête.

Si la closure est trouvée, on l'utilise, et on pense à la supprimer de la liste
pour ne pas conserver de closures inutiles.

```swift
class ImageLoader {

    var closuresByRequestId: [UUID: ((UIImage) -> Void)] = [:]

    func image(for url: URL, completionHandler: @escaping (UIImage) -> Void) -> UUID {

        let requestId = UUID()

        if let image = cache.object(forKey: url as NSURL) {

            completionHandler(image)

        } else {

            closuresByRequestId[requestId] = completionHandler

            downloadImage(from: url, requestId: requestId)
        }

        return requestId
    }

    func donwloadImage(from url: URL, requestId: UUID) -> Void) {

        URLSession.shared.dataTask(with: url) { data, response, error in
            
            let image = UIImage(data: data!)!
            
            self.cache.setObject(image, forKey: url as NSURL)

            if let completionHandler = closuresByRequestId[requestId] {

                completionHandler(image)

                closuresByRequestId.removeValue(forKey: requestId)
            }

        } .resume()
    }
}
```

Nous pouvons maintenant implémenter l'annulation :


```swift
class ImageLoader {

    ...

    func cancelRequest(with id: UUID) {

        closuresByRequestId.removeValue(forKey: id)
    }
}
```

Et voilà ! C'est tout ! Quand la méthode `donwloadImage(from:requestId:)` essayera
d'appeler la closure qui correspond à la requête, elle ne la trouvera pas dans la liste,
mais l'image aura tout de même été téléchargée et sera disponible pour une prochaine
utilisation.


# Implémentation de la cellule

Dans notre cellule, nous pouvons désormais annuler la dernière requête de chargement
d'image à chaque fois que la valeur de la propriété `imageURL` est mise à jour.
Nous réinitialisons également l'image avant toute nouvelle requête pour garantir
que seule l'image voulue est affichée.


```swift
class MyCell: UITableViewCell {

    var latestRequestId: UUID?

    var imageURL: URL {
        didSet {

            if let id = latestRequestId {
                ImageLoader.shared.cancelRequest(with: id)
            }

            mainImageView.image = nil

            latestRequestId = ImageLoader.shared.image(for: imageURL) { image in 
                DispatchQueue.main.async {
                    self.mainImageView.image = image
                }
            }
        }
    }
}
```


# Nettoyage correct de la cellule

Pour assurer une réinitialisation correcte de la cellule en cas de réutilisation,
nous pouvons utiliser la méthode `prepareForReuse()`. Cette méthode est appélée
par le système pour nous informer du fait que la cellule s'apprète à être réutilisée.


```swift
class MyCell: UITableViewCell {

    ...

    override func prepareForReuse() {

        if let id = latestRequestId {
            ImageLoader.shared.cancelRequest(with: id)
        }

        mainImageView.image = nil
    }
}
```


# Code complet

```swift
class ImageLoader {

    static let shared = ImageLoader()

    let cache = NSCache<NSURL, UIImage>()

    var closuresByRequestId: [UUID: ((UIImage) -> Void)] = [:]

    func image(for url: URL, completionHandler: @escaping (UIImage) -> Void) -> UUID {

        let requestId = UUID()

        if let image = cache.object(forKey: url as NSURL) {

            completionHandler(image)

        } else {

            closuresByRequestId[requestId] = completionHandler

            downloadImage(from: url, requestId: requestId)
        }

        return requestId
    }

    func donwloadImage(from url: URL, requestId: UUID) -> Void) {

        URLSession.shared.dataTask(with: url) { data, response, error in
            
            let image = UIImage(data: data!)!
            
            self.cache.setObject(image, forKey: url as NSURL)

            if let completionHandler = closuresByRequestId[requestId] {

                completionHandler(image)

                closuresByRequestId.removeValue(forKey: requestId)
            }

        } .resume()
    }

    func cancelRequest(with id: UUID) {

        closuresByRequestId.removeValue(forKey: id)
    }
}
```

```swift
class MyCell: UITableViewCell {

    var latestRequestId: UUID?

    var imageURL: URL {
        didSet {

            if let id = latestRequestId {
                ImageLoader.shared.cancelRequest(with: id)
            }

            mainImageView.image = nil

            latestRequestId = ImageLoader.shared.image(for: imageURL) { image in 
                DispatchQueue.main.async {
                    self.mainImageView.image = image
                }
            }
        }
    }

    override func prepareForReuse() {

        if let id = latestRequestId {
            ImageLoader.shared.cancelRequest(with: id)
        }

        mainImageView.image = nil
    }
}
```


# Améliorations

Notre fournisseur ne gère pas le cas où plusieurs clients demandent la même image.
Si cela se produit, la closure du dernier client remplace celle du premier,
et la closure du premier n'est donc jamais appelée.

La solution est simple, il suffit de stocker une liste de closure pour chaque requête.
Je vous laisse cela en exercice ;)

Dans un prochain article nous verrons comment améliorer la robustesse du fournisseur
d'image dans un contexte multithread.