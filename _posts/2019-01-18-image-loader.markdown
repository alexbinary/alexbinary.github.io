---
layout: post
title:  "Chargement d'images avec cache et usage dans un table view"
date:   2019-01-18 16:28:38 +0100
categories: ios
---

Dans votre app iOS vous voulez charger des images à partir de leur URL.
Vous affichez ces images dans un table view.
Vous voulez offir la meilleur expérience à vos utilisateur et miniser le consommation de données réseau.

Dans cet article nous allons voir comment implémenter un mécanisme simple pour télécharger et mettre en cache des images pour un usage dans un table view.
Dans la première partie nous allons voir comment télécharger les images et implémenter un cache.
Dans la deuxième partie nous procèderons à des améliorations pour l'usage dans un table view.


# Télécharger une image

Commençons par le coeur du sujet: télécharger une image depuis son URL.
On utilise [URLSession](https://developer.apple.com/documentation/foundation/urlsession) pour télécharger l'image,
et [UIImage.init?(data: Data)](https://developer.apple.com/documentation/uikit/uiimage/1624106-init) pour crééer une image à partir des données téléchargées:

```swift
let url = "https://www.pexels.com/photo/grey-and-white-short-fur-cat-104827/"

URLSession.shared.dataTask(with: url) { data, response, error in
            
    let image = UIImage(data: data!)!
    
    // utiliser l'image

} .resume()
```


# Un fournisseur d'image

On crée un objet qui sera responsable de fournir une image à partir de son URL.
Commençons par définir l'interface publique.

Notre classe définit une instance statique `shared` qui permet d'accéder facilement
à une instance unique dans toute l'application.

Notre classe a une méthode qui prend en premier paramètre l'URL de l'image à récupérer,
et en second paramètre une closure qui sera appelée une fois l'image prête.

```swift
class ImageLoader {

    static let shared = ImageLoader()

    func image(for url: URL, completionHandler: @escaping (UIImage) -> Void) {

        // todo
    }
}
```


# Utilisation dans le table view

On définit une classe qui hérite de UITableViewCell pour nos cellules.
La cellule possède une propriété `imageURL` qui permet de définir l'url de l'image que la cellule doit afficher.
Lorsqu'on set l'url, on met à jour l'image view de la cellule avec l'image correspondante.

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


# Le cache

Vu que les cellules d'un table view sont détruites et réutilisées dès qu'elles sortent de l'écran,
chaque image est redemandée à chaque fois que la cellule correspondante redevient visible.
Pour éviter de télécharger l'image à chaque fois, notre fournisseur d'image va mettre les images en cache.
Si on demande une image qui est déjà téléchargée, l'image en cache sera renvoyée.

On utilise `NSCache` pour notre cache. `NSCache` fonctionne avec un système de clé/valeur.
Les clés et les valeurs doivent descendre de `AnyObject`, ce qui n'est pas le cas de `URL`,
et c'est pour cela qu'on convertit les objets url en `NSURL`.

Chaque instance de `ImageLoader` possède son propre cache,
il est donc nécessaire de toujours utiliser la même instance pour profiter du cache.

Voilà la classe complète jusqu'ici:

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

Dans la deuxième partie nous procèderons à des améliorations pour l'usage dans un table view.