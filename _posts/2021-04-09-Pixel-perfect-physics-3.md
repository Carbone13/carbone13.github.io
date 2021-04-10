---
layout: post
categories: 2D Physic
title:  "[3/3] Réaliser sa propre physique pour du Pixel Art"
published: false
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

Nous allons commencer par crée cette classe `Solid` qui contient un rectangle en guise de boîte de collision :
```csharp
public abstract class Solid : MonoBehaviour
{
	public Rectangle collider;
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
	int leftEdgeSolid = solid.Bounds.xMin;
	int rightEdgeSolid = solid.Bounds.xMax;

	int ourBottom = collider.Bounds.yMin;
	int topEdgeSolid = solid.Bounds.yMax;

	bool isBetweenLeftAndRight = transform.position.x > leftEdgeSolid && transform.position.x < rightEdgeSolid;
	bool isTouchingY = (ourBottom == topEdgeSolid);

	return isTouchingY && isBetweenLeftAndRight;
}
```
