---
layout: post
categories: 2D Physic
title:  "[3/3] Réaliser sa propre physique pour du Pixel Art"
published: true
---

# Les Solides
---

Maintenant que nous en avons terminé avec les Acteurs, il est temps de s'attaquer au deuxième type d'entité : les Solides.
Ce que j'appelle un Solide, c'est une entité qui se déplace de façon impartiale, sans prendre en compte sans environnement.
La seule chose qu'un Solide fait, c'est de pousser les Acteurs sur son chemin, on alors de déplacer les Acteurs avec lui s'ils remplissent certaines conditions
Dans le cas d'un platformer ces conditions seraient simple :
- L'Acteur touche le Solide
- L'Acteur est au dessus du Solide
Ce qui se traduit par "L'Acteur est posé sur le Solide", dans ce cas, le Solide doit déplacer l'Acteur avec lui.
{: .text-justify}

Nous allons commencer par crée cette classe `Solid` qui contient un rectangle en guise de boîte de collision ainsi que le remainder que l'on va aussi cacher avec HideInInspector :
```csharp
public abstract class Solid : MonoBehaviour
{
	public Rectangle collider;

	[HideInInspector]
	public Vector2 remainder;
}
```

## Préparation des acteurs
---

Comme dit plus haut, les Acteurs doivent pouvoir vérifier s'il "chevauche" un Solide, pour que ce dernier sache s'il doit le déplacement avec lui.
Nous allons donc ajouter une fonction `IsRiding (Solid solid)` qui retourne une boolean dans notre class `Actor`. Cette fonction utilise le keyword `abstract` cela veut dire que
chaque classe qui hérite d'Actor va devoir fournir sa propre version de cette fonction
```csharp
public abstract bool IsRiding (Solid solid);
```

Une fois cela fait, Unity nous indique que notre classe Player ne peut pas compiler car elle n'implémente par `IsRiding()`, c'est ce que nous allons faire.
Il va falloir ajouter cette fameuse fonction dans notre Player, en utilisant le keyword `override`
```csharp
public override bool IsRiding (Solid solid)
{
	// Conditions
}
```

Pour nos test, nous allons définir quelque condition pour être considérer comme chevauchant :
- Le centre de notre boîte doit être compris entre les extrémités gauche et droite du Solide sur l'axe X
- Le bas de notre boîte doit être au même coordonnées Y que le haut du Solid

En code voilà ce que ça donne :
```csharp
public override bool IsRiding (Solid solid)
{
	int leftEdgeSolid = solid.collider.Bounds.xMin;
	int rightEdgeSolid = solid.collider.Bounds.xMax;

	int ourBottom = collider.Bounds.yMin;
	int topEdgeSolid = solid.collider.Bounds.yMax;

	bool isBetweenLeftAndRight = transform.position.x > leftEdgeSolid && transform.position.x < rightEdgeSolid;
	bool isTouchingY = (ourBottom == topEdgeSolid);

	return isTouchingY && isBetweenLeftAndRight;
}
```

Enfin, nous allons ajouter de la même manière une fonction `Squish()` à nos Acteurs.
Cette fonction sera appelé lorsqu'un Solide pousse un Acteur dans une autre entité, dans ce cas là
l'Acteur en question se retrouve alors écrasé entre 2 colliders.
```csharp
public abstract void Squish ();
```

Il faut donc aussi l'implémenter dans `Player` :
```csharp
public override void Squish ()
{
	print("Ouille !");
}
```
Vous pouvez mettre ce que vous voulez dans cette fonction, généralement soit tuer le joueur ou alors le pousser pour l'extraire.

## Petit ajout au Trackeur
Pour que nous Solide puisse récupérer les Acteurs qui le chevauche, il faut déjà accéder à tout les acteurs de la scène, pour se faire nous allons rajouter une Liste à notre Tracker qui stockera des Acteurs.
```csharp
public static List<Actor> actors = new List<Actor>();
```
Pour finir il ne reste plus qu'à peupler cette liste, de la même manière qu'avec les Rectangles en utilisant les fonctions OnEnable() et OnDisable() dans notre classe `Actor`.
```csharp
public void OnEnable ()
{
	Tracker.actors.Add(this);
}

public void OnDisable ()
{
	Tracker.actors.Remove(this);
}
```

## Les choses sérieuses
Maintenant que nous avons préparé nos Acteurs, retournons à notre classe Solid.
De la même façon que les Acteurs, pour se déplacer il va lui falloir un Vector2 en guise de remainder et une fonction `Move()`. Cette dernière est quasiment identique à celle de la classe Actor. Juste avant il va nous falloir un moyen de récupérer chaque Acteur qui nous chevauche, on va ajouter un HashSet d'Acteur, c'est similaire à une Liste mais chaque élement ne peut apparaître qu'une seule fois, et c'est bien plus performant qu'une Liste pour récuperer des éléments à chaque frames.
```csharp
public HashSet<Actor> riders = new HashSet<Actor>();
```

La fonction qui va permettre de la mettre à jour est simple, on loop à travers chaque Actor, on vérifie s'il nous chevauche d'après sa méthode `IsRiding()`, si c'est le cas on l'ajoute à notre HashSet.
```csharp
private void GetRiders ()
{
	riders.Clear();

	foreach (Actor actor in Tracker.actors)
	{
		if (actor.IsRiding(this))
		{
			riders.Add(actor);
		}
	}
}
```

Maintenant venons-en à la fonction Move(), comme je l'ai dit elle est quasiment identique à celle d'un Acteur, on commence déja par récupérer les acteurs nous chevauchant via GetRiders(), on désactive notre collider pour éviter de coincer les Acteurs en nous, on calcule le remainder comme sur un Acteur et enfin on réactive notre collider.
(Je continue de seulement détailler MoveX, le code complet est à la fin de l'article)
```csharp
protected void Move (Vector2 amount)
{
		GetRiders();

		collider.collidable = false;

		MoveX(amount.x);
		MoveY(amount.y);

		collider.collidable = true;
}

private void MoveX (float amount)
{
		remainder.x += amount;
		int toMove = (int) Math.Round(remainder.x);

		if (toMove != 0)
		{
				remainder.x -= toMove;
				PixelMoveX(toMove);
		}
}
```

Pour finir PixelMove() est un peu différente, nous n'avons pas besoin de vérifier de collision car comme dit en introduction, les solides ont un mouvement impartiales, ils sont garantis d'arriver à la position voulues. Nous allons donc commencer par déplacer notre transform.
Ensuite on loop à travers chaque Acteur. Il y a deux cas de figure:
- Soit on collide avec cet Acteur (une fois notre transform déplacée)
	--> Il faut pousser cet Acteur pour qu'il ne collide plus avec nous.
- Soit cet acteur nous chevauche.
 	--> Il faut lui appliquer le même mouvement que le notre.

Pousser l'acteur pour le sortir de notre boîte de collision consiste à le "snapper" au bord de notre collider en fonction de la direction dans laquelle nous allons. Pour se faire on le déplace de la différence entre nos 2 bords. Le callback est Squish, car si on se déplacant l'Acteur touche quelque chose, c'est qu'il est coincé entre nous et ce quelque chose.
Lui appliquer le même mouvement consiste à appeller sa fontion `PixelMove` avec comme argument `amount`.
```csharp
private void PixelMoveX (int amount)
{
	transform.position = new Vector2(transform.position.x + amount, transform.position.y);

	foreach (Actor actor in  Tracker.actors)
	{
		if(collider.Bounds.Overlaps(actor.collider.Bounds))
		{
			actor.remainder.x = 0;

			if (amount > 0)
				actor.MoveX(collider.Bounds.xMax - actor.collider.Bounds.xMin, actor.Squish);
			else
				actor.MoveX(collider.Bounds.xMin - actor.collider.Bounds.xMax, actor.Squish);
		}
		else if (riders.Contains(actor))
		{
			actor.PixelMoveX(amount, null);
		}
	}
}
```
À savoir qu'un Acteur ne peut pas subir les deux cas de figures en même temps
S'il nous chevauche et nous pénètre à la même frame, il est poussé (cas n°1).

La classe Solid complète :
```csharp
using System.Collections.Generic;
using UnityEngine;
using System;

public abstract class Solid : MonoBehaviour
{
	public Rectangle collider;
	[HideInInspector]
	public Vector2 remainder;

	public HashSet<Actor> riders = new HashSet<Actor>();

	protected void Move (Vector2 amount)
	{
		GetRiders();

		collider.collidable = false;

		MoveX(amount.x);
		MoveY(amount.y);

		collider.collidable = true;
	}

	private void MoveX (float amount)
	{
		remainder.x += amount;
		int toMove = (int) Math.Round(remainder.x);

		if (toMove != 0)
		{
				remainder.x -= toMove;
				PixelMoveX(toMove);
		}
	}

	private void MoveY (float amount)
	{
		remainder.y += amount;
		int toMove = (int) Math.Round(remainder.y);

		if (toMove != 0)
		{
				remainder.y -= toMove;
				PixelMoveY(toMove);
		}
	}

	private void PixelMoveX (int amount)
	{
		transform.position = new Vector2(transform.position.x + amount, transform.position.y);

		foreach (Actor actor in  Tracker.actors)
		{
			if(collider.Bounds.Overlaps(actor.collider.Bounds))
			{
				actor.remainder.x = 0;

				if (amount > 0)
					actor.MoveX(collider.Bounds.xMax - actor.collider.Bounds.xMin, actor.Squish);
				else
					actor.MoveX(collider.Bounds.xMin - actor.collider.Bounds.xMax, actor.Squish);
			}
			else if (riders.Contains(actor))
			{
				actor.PixelMoveX(amount, null);
			}
		}
	}

	private void PixelMoveY (int amount)
	{
		transform.position = new Vector2(transform.position.x, transform.position.y + amount);

		foreach (Actor actor in  Tracker.actors)
		{
			if(collider.Bounds.Overlaps(actor.collider.Bounds))
			{
				actor.remainder.y = 0;

				if (amount > 0)
					actor.MoveY(collider.Bounds.yMax - actor.collider.Bounds.yMin, actor.Squish);
				else
					actor.MoveY(collider.Bounds.yMin - actor.collider.Bounds.yMax, actor.Squish);
			}
			else if (riders.Contains(actor))
			{
				actor.PixelMoveY(amount, null);
			}
		}
	}

	private void GetRiders ()
	{
		riders.Clear();

		foreach (Actor actor in Tracker.actors)
		{
			if (actor.IsRiding(this))
			{
				riders.Add(actor);
			}
		}
	}
}
```

Pour utiliser cette class je vous propose de réaliser des plateformes qui se déplacent
Vu que ce n'est pas le but du tutoriel et que l'utilisation est la même qu'avec Actor, voici un script réalisé par Sebastian Lague :
```csharp
public class MovingPlatform : Solid
{
    public Vector3[] localWaypoints;
    Vector3[] globalWaypoints;

    public float speed;
    public bool cyclic;

    private int fromWaypointIndex;
    private int toWaypointIndex;


    private void Start ()
    {
        globalWaypoints = new Vector3[localWaypoints.Length];
        for (int i = 0; i < localWaypoints.Length; i++)
        {
            globalWaypoints[i] = localWaypoints[i] + transform.position;
        }

    }

    private void Update ()
    {
        fromWaypointIndex %= globalWaypoints.Length;
        toWaypointIndex = (fromWaypointIndex + 1) % globalWaypoints.Length;

        Vector2 dir = (globalWaypoints[toWaypointIndex] - globalWaypoints[fromWaypointIndex]).normalized;

        Move(dir * speed * Time.deltaTime);

        if (transform.position == globalWaypoints[toWaypointIndex])
        {
            remainder = Vector2.zero;
            fromWaypointIndex ++;

            if (!cyclic)
            {
                if (fromWaypointIndex >= globalWaypoints.Length-1) {
                    fromWaypointIndex = 0;
                    Array.Reverse(globalWaypoints);
                }
            }
        }
    }

    public void OnDrawGizmos ()
    {
        if (localWaypoints != null)
        {
            Gizmos.color = Color.red;
            float size = 1f;

            for (int i = 0; i < localWaypoints.Length; i++)
            {
                Vector3 globalWaypointPosition = (Application.isPlaying) ? globalWaypoints[i] : localWaypoints[i] + transform.position;
                Gizmos.DrawLine(globalWaypointPosition - Vector3.up * size, globalWaypointPosition + Vector3.up * size);
                Gizmos.DrawLine(globalWaypointPosition - Vector3.left * size, globalWaypointPosition + Vector3.left * size);
            }
        }
    }
}
```

Résultat final sur Unity :
![Helena se fait brutaliser par un solide](/assets/pixelartphysic/solid.gif)
