---
layout: post
categories: 2D Physic
title:  "Réaliser sa propre physique pour du Pixel Art [2/3]"
---

## Les acteurs



Lorsqu'on souhaite réaliser un jeu en 2D sur Unity, on peut se laisse tenter par son système de Physic et de collisions.
Malheuresement il est assez laborieux d'avoir un résultat convaincant avec, surtout lorsqu'on développe un Platformer.
Maintenant il est toujours possible d'utiliser la fonction MovePosition() d'un rigidbody, mais le système de collision est assez strict.

C'est pourquoi aujourd'hui je vais vous apprendre à réaliser vous même un système de collision.
Ce dernier est inspiré du celébre jeu Céleste. Avant de commencer il y a quelque contrainte à prendre en compte :
Chaque Acteur ne peut se déplacer que d'un pixel par un pixel.
Chaque boîte de collision est forcément une AABB (**A**xis **A**ligned **B**ounding **B**ox), c'est à dire un rectangle non tourné.
Ces boîtes de collisions ont des dimensions en pixels, pas de virgule admise.


# Les boîtes de collisions
Comme je viens de le dire, la seule forme que notre système va pouvoir gérer sont les rectangles.
Il faut donc bien comprendre qu'il faut faire une croix sur les pentes ou autres forme farfelues.

Pour être conforme à ses boîtes et au système en général, nos sprites vont devoir respecter certaines régles :
- Leur pivot doit être positionner sur un pixel du sprite, et pas entre 2 pixels.
- Leur PPU (**P**ixel **p**er **U**nit) doit être à 1, ainsi 1 pixel de notre sprite sera égal à 1 unité sur Unity.
- Optionnel mais reccommandé pour du pixel art : Désactiver la compression ("None") et passer le Filter Mode en "Point"

Maintenant passons aux choses sérieuses. Pour répresenter nos boîtes nous allons crée une classe nommée "Rectangle.cs".
En soit je préfère l'appeler 'AABB' mais l'éditeur Unity la fait entrer en conflit avec une classe du même nom.
Nous n'allons pas tout faire de A à Z, il existe déja une structure permettant de répresenter un carré avec des dimensions
en pixels (rappelez vous, contraintes n°3), il s'agit de RectInt !

Pour une meilleure lisibilité, j'aime ajouté 2 `Vector2Int`, un pour la taille et un pour renseigner un offset.
Ces deux variables ne sont pas accesibles depuis d'autre script, elles sont privés mais sérialisés.

J'ajoute aussi une `boolean` "Collidable" qui permet de désactiver la boîte si besoin.
```csharp
[SerializeField] private Vector2Int size;
[SerializeField] private Vector2Int offset;

public bool Collidable = true;
```

Maintenant, les autres scripts doivent pouvoir récuperer un `RectInt` qui correspond à notre boite.
Nous allons utiliser une variable avec un simple getter qui va permettre de crée ce Rect.
Il faut aussi penser à prendre en compte la position de l'objet dans la scène.
```csharp
public RectInt Bounds => new RectInt(transform.position.v2i() + Offset, Size);
```

Vous remarquez que j'utilise la fonction `v2i()` sur mon Vector3, c'est une fonction que j'ai crée qui permet de convertir un
Vector3 en Vector2Int, car rappelez vous, nous travaillons en pixel entier donc avec des nombres entiers.

Pour les intéresser la voici (à mettre dans une classe statique elle aussi) :
```csharp
public static Vector2Int v2i (this Vector3 input)
{
	return new Vector2Int(Mathf.RoundToInt(input.x), Mathf.RoundToInt(input.y));
}
```

Pour terminer nos boîtes, j'aimerais ajouter une petite fonction de Debug permettant de les visualiser.
Pour se faire, j'utilise la fonction DrawWireCube de Gizmos, qui permet de dessiner les arêtes d'un rectangle.
Le rectangle étant donc notre RectInt "bounds"
```csharp
public void OnDrawGizmos ()
{
	Gizmos.DrawWireCube(Bounds.center, new Vector2(Bounds.size.x, Bounds.size.y));
}
```

Notre classe rectangle ressemblant donc au final à ça :
```csharp
public class Rectangle : MonoBehaviour
{
	[SerializeField] private Vector2Int size;
	[SerializeField] private Vector2Int offset;

	public bool collidable = true;

	public RectInt Bounds => new RectInt(transform.position.v2i() + offset, size);


	// Editor Functions
	public void OnDrawGizmos ()
	{
		Gizmos.color = Color.green;
		Gizmos.DrawWireCube(Bounds.center, new Vector2(Bounds.size.x, Bounds.size.y));
	}
}
```

Résultat final en scène :
![Héléna avec sa boîte de collision](/assets/pixelartphysic/HelenaWithCollider.jpg)
