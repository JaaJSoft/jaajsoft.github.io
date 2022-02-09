---
layout: article
title: "Créer ses propres références en LaTeX"
author: Louis Chopinet
tags:
- LaTeX
- tutoriel

---

Dans ce tutoriel, nous allons apprendre comment créer son propre système de
références et d'étiquettes en $\LaTeX$, parallèle à ceux des sections, des
tableaux ou des équations par exemple. <!--more-->

## Introduction : étiquettes et références

Il n'y a besoin d'aucun *package* particulier : on étiquette un endroit du
document avec un `\label{mon label unique}`, puis pour faire référence à cet
endroit du document à un autre endroit, on utilise `\ref{mon label unique}`. Par
exemple, le code :

```latex
\documentclass{article}

\begin{document}
\section{Première section}
\subsection{Une sous-section qui servira ensuite}
\label{un label unique}
Cette sous-section très importante sera réutilisée plus tard...

\section{Blabla}
Blabla

\section{Beaucoup plus...}
... loin dans le document, je peux faire une référence à \ref{un label unique}.

\end{document}
```

donne le résultat :
![image 1](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image1.jpg)

Si vous utilisez le *package* `hyperref`, dont nous reparlerons, ces références
sont transformées, dans un pdf, en liens vers l'endroit où vous avez placé votre
étiquette. On obtient, toutes choses égales par ailleurs :
![image 2](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image2.jpg)

## Prérequis : utilisation des compteurs

En fait, chaque type de référence (section, figure, table ou équation par
exemple) utilise un compteur, qui sert à dénombrer le nombre de sections, de
figures, de tables ou d'équations. Lorsque vous écrivez `\section{...}`, le
compteur de sections est incrémenté : on fait habituellement cela en
utilisant `\\stepcounter{moncompteur}`. Ainsi, si l'on souhaite créer une
nouvelle commande `\encadre`qui créer un cadre de texte, commençant par quelque
chose que "Encadré n°3", on aura tout intérêt à définir un compteur `cntEncadre`
qui sera incrémenté automatiquement par `\encadre`, et qu'on affichera après
le "Encadré n°". C'est ce que fait le code suivant :

```latex
\newcounter{encadre}
\newcommand{\encadre}[1]{\stepcounter{encadre}%
	\medskip%
	\noindent\hspace{-\fboxsep}\fbox{%
		\parbox{\linewidth}{%
			\textbf{Encadré n°\theencadre~:} #1
		}%
	}%
	\medskip%
}
```

Le fonctionnement précis de cette commande n'est pas le sujet de cet article,
seuls nous intéressent l'appel à `\stepcounter` et `\theencadre`. Le code :

```latex
\blindtext

\encadre{Bonjour, je suis un encadré, le premier d'entre eux.}

\blindtext

\encadre{Bonjour, je suis le deuxième encadré.}

\end{document}
```

donne le résultat :
![image 3](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image3.jpg)

## Utiliser `\refstepcounter` pour créer une étiquette

Nous allons utiliser des compteurs pour créer nos propres
étiquettes. `\refstepcounter` fonctionne de la même manière que sa
cousine `\stepcounter`, seulement elle place une sorte d'ancre là où elle a été
appelée afin d'y avoir accès ailleurs dans le document, par exemple lors de
l'appel à un `\ref`. Par défaut, cet appel à `\ref` renverra la valeur qu'avait
le compteur quand elle a été placé. Si nous remplaçons `\stepcounter{encadre}`
par `\refstepcounter{encadre}` dans le code précédent, tout fonctionne très
bien, mais le document pdf contiendra désormais une ancre au début de chaque
encadré. Pour aller chercher cette ancre, nous avons besoin de la nommer par une
sorte d'identifiant unique. C'est ce qui est fait par `\label{identifiant}` : il
nomme la dernière ancre placée "identifiant". Il ne reste plus alors qu'à
appeler `\ref{identifiant}`, qui renverra la valeur du dernier compteur
incrémenté par `\refstepcounter` avant le `\label`.

Ainsi, le code :

```latex
\documentclass{article}
\usepackage{blindtext}

\newcounter{encadre}
\newcommand{\encadre}[1]{\refstepcounter{encadre}%
	\medskip%
	\noindent\hspace{-\fboxsep}\fbox{%
		\parbox{\linewidth}{%
			\textbf{Encadré n°\theencadre~:} #1
		}%
	}%
	\medskip%
}

\begin{document}

\blindtext

\encadre{\label{premier des encadres}Bonjour, je suis un encadré, le premier d'entre eux.}

\blindtext

\encadre{\label{une étiquette différente !}Bonjour, je suis le deuxième encadré.}

Nous avons vu dans l'encadré \ref{premier des encadres} que blabla.

Cela a été approfondi dans l'encadré \ref{une étiquette différente !}.

\end{document}
```

donne le résultat :
![image 3](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image3.jpg)

*Remarque* : nous écrivons toujours "encadré" avant d'appeler `\ref`, mais il
est toujours possible, et conseillé, d'envelopper cet appel dans une commande *
sémantique*, comme :

```latex
\newcommand{\refEncadre}[1]{%
	\textcolor{blue}{\emph{encadré n°\ref{#1}}}%
}
```

## Utilisation de `hyperref` pour créer des liens

Le *package* `hyperref`permet de transformer les `\ref` (ainsi que ses petits
frères comme le `\eqref` défini par `amsmath`) en liens. Cette fonctionnalité du
format pdf permet de naviguer rapidement dans le document jusqu'à l'endroit
auquel on fait référence (la fameuse ancre dont nous parlions précédemment).
C'est particulièrement utile dans la table des matières par exemple. Il suffit
d'inclure `\usepackage{hyperref}` dans le préambule pour que la magie opère. Il
vaut cependant mieux l'appeler *en dernier*, car il fonctionnera ainsi dans le
plus grand nombre de cas possibles (ceux du noyau de $\LaTeX$ comme les figures
ou les sections, mais aussi ceux rajoutés par des *packages* comme les
environnements de `amsmath`).

Le même code que précédemment, auquel on rajoute cet appel à `hyperref` donne
ainsi le résultat :
![image 5](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image5.jpg)

Cliquer sur les parties encadrées en rouge mène directement jusqu'aux encadrés
en question (il est possible de changer l'apparence des liens, *cf.* la document
de `hyperref`).

Cela est particulièrement utile lorsqu'on utilise des notions complexes qui ont
été définies à un autre endroit du document : le lecteur pourra rapidement
revenir à leur définition. Pour cela, il peut être intéressant de raffiner un
peu notre système de référence pour que le lien soit plus clair qu'un simple
numéro. Pour des références "classiques", `hyperref`
définit `\autoref{identifiant}`, qui fonctionne comme `\ref` mais ajoute devant
le numéro (pour rappel, ce numéro n'est rien d'autre que la valeur du compteur
utilisé) un terme comme "section", "figure" ou "équation", selon le contexte
dans lequel a été appelé `\label` (juste après une section, dans une figure ou
dans une équation par exemple).

Pour notre cas, on utilisera
plutôt `\hyperref[identifiant]{texte de remplacement}` qui fonctionne exactement
comme `\ref{identifiant}`, à cela près que le texte affiché est "texte de
remplacement" et non la valeur du compteur utilisé.
Utiliser `\hyperref[une étiquette différente !]{cet encadré}` permet
d'utiliser "cet encadré" comme lien :

![image 5](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image5.jpg)

Cela permet d'écrire des documents faciles à lire et à comprendre. En utilisant
les options de `hyperref` ou en définissant des macros sémantiques qui
appellent (par exemple) `\hyperref[#1]{#2}` après un formatage adéquat du texte,
on peut définir à un endroit :

![image 6](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image6.jpg)

et y faire référence de manière très claire à un autre endroit :

![image 7](/assets/images/2022-02-06-Créer-ses-propres-références-en-latex/image7.jpg)

## Conclusion

Comme la plupart des _packages_ $\LaTeX$, la meilleure manière d'apprendre à
utiliser `hyperref` est de tester constamment de nouvelles manières de faire

Adaptez ce qui précède à vos nouveaux documents : ce qui a marché pour des
encadrés, marchera pour des définitions ce qui a marché pour des définitions
peut servir à transformer une variable mathématique en lien l'endroit où elle a
été fixé par exemple !

Et si cela ne marche pas, il faut n'y voir qu'une occasion de mieux comprendre
`hyperrref`.

## Voir aussi

- [Documentation de hyperref](https://ctan.org/pkg/hyperref)
- [GitHub de hyperref](https://github.com/latex3/hyperref)
