---
layout: post
categories: 2D Physic
title:  "[1/3] Réaliser sa propre physique pour du Pixel Art"
---


# Intro

---
Lorsqu'on souhaite réaliser un jeu en 2D sur Unity, on peut se laisser tenter par son système de Physique et de Collisions.
Malheureusement il est assez laborieux d'avoir un résultat convaincant avec, surtout lorsqu'on développe un Platformer.
Maintenant il est toujours possible d'utiliser la fonction MovePosition() d'un rigidbody, mais le système de collision est assez strict.
{: .text-justify}

C'est pourquoi aujourd'hui je vais vous apprendre à réaliser vous même un système de collision.
Ce dernier est inspiré du célèbre jeu Céleste. Avant de commencer il y a quelque contrainte à prendre en compte :
- Chaque Acteur ne peut se déplacer que d'un pixel par un pixel.
- Chaque boîte de collision est forcément une AABB (**A**xis **A**ligned **B**ounding **B**ox), c'est à dire un rectangle non tourné.
- Les tailles des boîtes ainsi que les positions de chaque entités sont des nombres entiers.
{: .text-justify}

Les scripts complets sont disponibles à la fin de chaque articles.
Les sources complètes des projets sont disponibles à la fin de chaque série d'article compressé en .zip
{: .text-justify}

# Les boîtes de collisions

---

Comme je viens de le dire, la seule forme que notre système va pouvoir gérer sont les rectangles.
Il faut donc bien comprendre qu'il faut faire une croix sur les pentes ou autres forme farfelues.
{: .text-justify}

Pour être conforme à ses boîtes et au système en général, nos sprites vont devoir respecter certaines règles :
- Leur pivot doit être positionner sur un pixel du sprite, et pas entre 2 pixels.
- Leur PPU (**P**ixel **p**er **U**nit) doit être à 1, ainsi 1 pixel de notre sprite sera égal à 1 unité sur Unity.
- Optionnel mais recommandé pour du pixel art : Désactiver la compression ("None") et passer le Filter Mode en "Point"
{: .text-justify}

Maintenant passons aux choses sérieuses. Pour représenter nos boîtes nous allons crée une classe nommée "Rectangle.cs".
En soit je préfère l'appeler 'AABB' mais l'éditeur Unity la fait entrer en conflit avec une classe du même nom.
Nous n'allons pas tout faire de A à Z, il existe déjà une structure permettant de représenter un carré avec des dimensions
en pixels (rappelez vous, contraintes n°3), il s'agit de RectInt !
{: .text-justify}

On commence par déclarer 2 Vector2Int qui serviront à renseigner une taille en pixel ainsi qu'un offset lui aussi en pixel.
{: .text-justify}
J'ajoute aussi une `boolean` "Collidable" qui permet de désactiver la boîte si besoin.
{: .text-justify}
```csharp
public class Rectangle : MonoBehaviour
{
	public Vector2Int size;
	public Vector2Int offset;

	public bool Collidable = true;
}
```

Maintenant, les autres scripts doivent pouvoir récupérer un `RectInt` qui correspond à notre boite dans la scène.
Nous allons donc ajouter une variable avec un simple getter qui va permettre de "générer" ce Rect.
Il faut aussi penser à prendre en compte la position de l'objet dans la scène.
{: .text-justify}
```csharp
public RectInt Bounds => new RectInt(transform.position.v2i() + Offset, Size);
```

Vous remarquez que j'utilise la fonction `v2i()` sur mon Vector3, c'est une fonction que j'ai crée qui permet de convertir un
Vector3 ou en Vector2 en Vector2Int, car rappelez vous, nous travaillons en pixel entier donc avec des nombres entiers.
{: .text-justify}
Pour les intéresser la voici (à mettre dans une classe statique elle aussi) :
{: .text-justify}
```csharp
public static Vector2Int v2i (this Vector3 input)
{
	return new Vector2Int(Mathf.RoundToInt(input.x), Mathf.RoundToInt(input.y));
}

public static Vector2Int v2i (this Vector2 input)
{
	return new Vector2Int(Mathf.RoundToInt(input.x), Mathf.RoundToInt(input.y));
}
```

Pour terminer nos boîtes, j'aimerais ajouter une petite fonction de Debug permettant de les visualiser.
Pour se faire, j'utilise la fonction `DrawWireCube()` de `Gizmos`, qui permet de dessiner les arêtes d'un rectangle.
Le rectangle étant donc notre `RectInt` "bounds"
{: .text-justify}
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
	public Vector2Int size;
	public Vector2Int offset;

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



# Le Trackeur

---

Maintenant que nous avons nos boîtes, il va falloir les tracker pour les retrouver facilement.
Pour détecter si une boîte A collisione avec une autre boîte dans la scène, nous n'avons pas d'autre choix
que de la tester contre toutes les autres. Si l'on considère une scène avec n acteurs, chaque acteur devras vérifier
s'il collisione avec chacun des autres acteurs, cela donne donc un problème de type O(n²). Nous verrons dans un futur poste
un moyen d'optimiser tout ça.
{: .text-justify}

Revenons en à notre Tracker, j'ai décidé d'utiliser une classe statique qui n'est pas instancié sur Unity, mais vous pouvez très bien
la faire dériver de `MonoBehaviour` et l'ajouter sur un GameObject "Manager".
Cette classe ne possède qu'une liste de Rectangles accessible à tous :
{: .text-justify}
```csharp
public static class Tracker
{
	public static List<Rectangle> colliders = new List<Rectangle>();
}
```

Il ne reste plus qu'à ajouter nos boîtes dans cette liste, pour se faire j'utilise les fonctions
`OnEnable()` et `OnDisable()` de `MonoBehaviour` dans ma classe Rectangle :
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

	public void OnEnable ()
	{
		Tracker.colliders.Add(this);
	}

	public void OnDisable ()
	{
		Tracker.colliders.Remove(this);
	}
}
```

Vous pouvez maintenant ajouter des obstacles dans votre scène.
Pour leur ajouter des colliders, il suffit de leur ajouter le composant Rectangle.
Noter que vous n'êtes pas limité quant au nombre de Rectangle par Game Object.
{: .text-justify}
