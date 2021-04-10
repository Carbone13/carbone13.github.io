---
layout: post
categories: 2D Physic
title:  "[3/3] Réaliser sa propre physique pour du Pixel Art"
published: false
---

## Les Solides

Maintenant que nous en avons terminé avec les Acteurs, il est temps de s'attaquer au deuxième type d'entité : les Solides.
Ce que j'appelle un Solide, c'est une entité qui se déplace de façon impartiale, sans prendre en compte sans environnement.
La seule chose qu'un Solide fait, c'est de pousser les Acteurs sur son chemin, on alors de déplacer les Acteurs avec lui s'ils remplissent certaines conditions
Dans le cas d'un platformer ces conditions seraient simple :
- L'Acteur touche le Solide
- L'Acteur est au dessus du Solide
Ce qui se traduit par "L'Acteur est posé sur le Solide", dans ce cas, le Solide doit déplacer l'Acteur avec lui
{: .text-justify}

Personnellement je crée une class abstraite que j'appelle Actor. Abstraite signifie qu'elle ne peux pas être utiliser comme ça, il
faut forcément crée une deuxième classe qui en dérive pour l'utiliser. Le but ensuite, c'est de crée des fonctions dans la classe Actor
que tout les classes qui en hérite pourront utiliser.
{: .text-justify}

# Se déplacer pixel par pixel
Pour que nos Acteurs se déplacent pixel par pixel, il faut stocker le mouvement de ce dernier, est dès lors qu'il est d'au moins 1 pixel,
on déplace réellement le joueur dans la scène. Chaque Acteur possède donc une variable de type Vector2 que j'appelle remainder qui stock donc
les déplacements. Ensuite les Acteur possèdent une fonction `Move()` qui prend en paramètre un Vector2 qui représente la distance à parcourir.
Aussi il est important que chaque Acteur possède une boîte de collision
{: .text-justify}
```csharp
public abstract class Actor : MonoBehaviour
{
	protected Rectangle collider;
	protected Vector2 remainder;

	private void Awake ()
	{
		collider = GetComponent<Rectangle>();
	}

	protected void Move (Vector2 amount)
	{
		MoveX(amount.x);
		MoveY(amount.y);
	}
}
```

Ensuite chaque mouvement est divisé en 2 phase, on s'occupe d'abord de déplacement horizontalement le joueur, puis verticalement, axe par axe.
Ces deux fonctions sont les mêmes, alors pour le tutoriel je ne vais en détailler qu'une seul (les deux fonctions seront tout de même disponible à la fin de l'article).
Le principe est simple, déjà on vérifie que la distance à bouger n'est pas de zéro, si c'est n'est pas le cas, on l'ajoute dans notre remainder, on va ensuite arrondir le remainder à l'entier le plus proche. Si la valeur arrondie n'est pas zéro, alors on doit déplacer notre joueur d'un certain nombre de pixel. Noter que cette valeur peux être négative.
{: .text-justify}
```csharp
private void MoveX (float amount)
{
	if(amount == 0) return;

	remainder.x += amount;
	int toMove = (int) Math.Round(remainder.x);

	if(toMove != 0)
	{
		remainder.x -= toMove;
		PixelMoveX (toMove);
	}
}
```

Enfin, on appelle la fonction `PixelMove()`, qui prend en paramètre un int.
Cette dernière fonction simule chaque pixel un par un au lieu de les faire en même temps.
Cela permet de calculer les positions entre le point de départ et celui d'arrivée.
Le principe est simple, pour chaque déplacement d'un pixel, on regarde si l'on va collisioner un objet à la "position voulue".
Si oui on ne bouge pas, si non on déplace le transform du joueur.
L'opération est repétée pour chaque pixel.
{: .text-justify}
```csharp
private void PixelMoveX (int amount)
{
	int sign = Math.Sign(amount);

	while(amount != 0)
	{
		Rectangle hit = First(transform.position.v2i() + Vector2.right * sign);

		if(hit == null)
		{
			transform.position = transform.position.v2i() + Vector2.right * sign;
		}
		else
		{
			// Collision !
			break;
		}
	}
}
```

On retrouve par ailleurs la fonction v2i cité dans le premier article.
La dernière fonction qu'il reste à élucider est la fonction First
Elle permet de retourner le premier Rectangle touché par notre boîte à une position donnée.
Son fonctionnement est simple, elle vérifie chacune des autres boîtes présentes dans la scène.
On vérifie d'abord si collidable est true, et que la boîte n'est pas la notre.
Puis, si on collisione avec, on la retourne.
A savoir que la fonction `Overlaps()` provient de la structure RectInt.
{: .text-justify}
```csharp
private Rectangle First (Vector2 position)
{
	RectInt colliderAtPosition = new RectInt(position.v2i() + collider.offset, collider.size);

	foreach(Rectangle rect in Tracker.colliders)
	{
		if(rect.collidable && rect != collider)
		{
			if(colliderAtPosition.Overlaps(rect.Bounds))
			{
				return rect;
			}
		}
	}

	return null;
}
```

Voici Actor.cs dans sa version complète :
```csharp
public abstract class Actor : MonoBehaviour
{
	protected Rectangle collider;
	protected Vector2 remainder;

	private void Awake ()
	{
		collider = GetComponent<Rectangle>();
	}

	protected void Move (Vector2 amount)
	{
		MoveX(amount.x);
		MoveY(amount.y);
	}

	private void MoveX (float amount)
	{
		if(amount == 0) return;

		remainder.x += amount;
		int toMove = (int) Math.Round(remainder.x);

		if(toMove != 0)
		{
			remainder.x -= toMove;
			PixelMoveX (toMove);
		}
	}

	private void MoveY (float amount)
	{
		if(amount == 0) return;

		remainder.y += amount;
		int toMove = (int) Math.Round(remainder.y);

		if(toMove != 0)
		{
			remainder.y -= toMove;
			PixelMoveY (toMove);
		}
	}

	private void PixelMoveX (int amount)
	{
		int sign = Math.Sign(amount);

		while(amount != 0)
		{
			Rectangle hit = First(transform.position.v2i() + Vector2.right * sign);

			if(hit == null)
			{
				transform.position = transform.position.v2i() + Vector2.right * sign;
			}
			else
			{
				// Collision !
				break;
			}
		}
	}

	private void PixelMoveY (int amount)
	{
		int sign = Math.Sign(amount);

		while(amount != 0)
		{
			Rectangle hit = First(transform.position.v2i() + Vector2.up * sign);

			if(hit == null)
			{
				transform.position = transform.position.v2i() + Vector2.up* sign;
			}
			else
			{
				// Collision !
				break;
			}
		}
	}

	private Rectangle First (Vector2 position)
	{
		RectInt colliderAtPosition = new RectInt(position.v2i() + collider.offset, collider.size);

		foreach(Rectangle rect in Tracker.colliders)
		{
			if(rect.collidable && rect != collider)
			{
				if(colliderAtPosition.Overlaps(rect.Bounds))
				{
					return rect;
				}
			}
		}

		return null;
	}
}
```

N'oubliez pas qu'Actor est un classe abstraite, il faut forcément crée une classe qui en dérive pour l'utiliser.
Voici une classe Player basique pour vous montrer comment l'utiliser :
{: .text-justify}
```csharp
public class Player : Actor
{
	public float MoveSpeed = 75.0f;

	public void Update ()
	{
		float xInput = Input.GetAxisRaw("Horizontal");
		float yInput = Input.GetAxisRaw("Vertical");

		Vector2 inputs = new Vector2(xInput, yInput);

		Move(inputs.normalized * MoveSpeed * Time.deltaTime);
	}
}
```
On dérive d'abord d'Actor, ensuite je déclare une variable MoveSpeed qui correspond à la vitesse du joueur en pixel par seconde.
Dans une fonction Update je récupère les axes Horizontal et Vertical (qui correspondent aux flèche directionnelle ou à ZQSD).
{: .text-justify}
Enfin je déplacer mon joueur en fonction de mes touches (que je normalize pour transformer le tout en direction), je multiplie par ma vitesse
et par Delta Time (voir futur article sur le delta time).
{: .text-justify}
D'ailleurs j'ai oublier de le préciser, mais la position des joueurs doit être en chiffre entier. Sinon les boîtes seront de toute façon alignés
à la grille de pixel, mais les visuels seront décalés !
{: .text-justify}
Résultat final sur Unity :
![Héléna qui coure dans tout les sens](/assets/pixelartphysic/actortest.gif)
