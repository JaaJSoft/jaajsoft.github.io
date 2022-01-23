---
layout: article title: "Python : Comment sauvegarder des tableaux NumPy"
author: Pierre Chopinet tags:

- python
- numpy
- array
- persistence
- tutoriel

---

Dans ce tutoriel, nous allez apprendre à sauvegarder des tableaux numpy pour
ajouter de la persistence à votre application python. <!--more-->

## Notre premiere persistence

### Sauvegarde des données

Pour sauvegarder votre tableau NumPy on utilise le comportement natif de python
pour l'ouverture d'un fichier, et on appelle la function save de NumPy :

```python
array = np.array([6, 9])

with open('array.npy', 'wb') as f:
    np.save(f, array)
```

Le fichier en sortie utilise un format binaire spécifique numpy. Sur la premiere
ligne on trouve les paramètres de notre tableau persisté :

```
�NUMPYv{'descr': '<i8', 'fortran_order': False, 'shape': (2,), }
```

La suite du fichier est composé d'octets correspondant aux valeurs du tableau.

### Chargement des données

Après avoir sauvegardé nos données, il faut pouvoir les charger. Pour cela on
utilise la fonction `load` de numpy :

```python
with open('array.npy', 'rb') as f:
    array2 = np.load(f)
    print(array2)
```

Ce qui donne le résultat suivant :

```
[6 9]
```

Et voilà on a notre première persistence de données avec numpy !

## Persistence au format texte

Il arrive qu'on veuille utiliser des données traitées avec numpy avec un autre
langage de programmation, ou par exemple sur excel. On doit donc enregistrer nos
données dans un format compréhensible par tous.

### Sauvegarde des données

On utilise la function `savetxt` :

```python
array = np.array([[6, 9, 42], [4, 2, 9]])
np.savetxt('array_float.csv', array, delimiter=',')
```

Ce qui donne le résultat suivant :

```csv
6.000000000000000000e+00,9.000000000000000000e+00,4.200000000000000000e+01
4.000000000000000000e+00,2.000000000000000000e+00,9.000000000000000000e+00
```

Les nombres sont enregistrés par défaut en nombre flottant, pour régler ça, on
spécifie un format à notre export :

```python
np.savetxt('array_int.csv', array, delimiter=',', fmt='%i')
```

Finalement, on obtient un format en entier :

```csv
6,9,42
4,2,9
```

### Chargement des données

Maintenant pour charger nos données enregistrées en format texte, on utilise la
fonction `loadtxt` :

```python
array_loaded_from_text = np.loadtxt('array_int.csv', delimiter=',', )
print(array_loaded_from_text)
```

```
[[ 6.  9. 42.]
 [ 4.  2.  9.]]
```

### Compression en archive gz

Si la quantité de données est conséquente, il peut être intéressant de
compresser les données enregistrées. La fonction de sauvegarde utilisée
précédemment `savetxt` compressera nativement les données si le fichier de
sortie possède l'extension .gz, pour gzip ou GNU zip.

```python
big_array = np.random.rand(1000, 1000) # tableau de 1000x1000
np.savetxt('array_test.csv', big_array, delimiter=',', fmt='%i')
np.savetxt('array_compressed.gz', big_array, delimiter=',', fmt='%i')
```

| Fichier | Taille  |
|---------|---------|
| .csv    | 1954 Ko |
| .gz     | 4 Ko    |

Le gain de place est très intéressant, cependant l'enregistrement et le
chargement des données sauvegardées au format gz sera plus long.

Le chargement se fait de la meme façon que précédemment, la fonction `loadtxt`
se charge de décompresser pour nous les données.

```python
big_array_loaded = np.loadtxt('array_compressed.gz', delimiter=',')
```

## Persistence au format npz

NumPy propose une dernière façon de persister vos données avec le format npz qui
a l'avantage de pouvoir sauvegarder plusieurs tableaux numpy dans le même
fichier et contrairement au format texte, ce format supporte des tableaux à n
dimensions

```python
np.savez('test.npz', array=array, big_array=big_array)
loaded_npz = np.load('test.npz')
array_loaded_from_npz = loaded_npz['array']
big_array_loaded_from_npz = loaded_npz['big_array']

print(loaded_npz.files)
print(big_array_loaded_from_npz.shape)
```

Ce qui donne :

```
['array', 'big_array']
(1000, 1000)
```

De plus, NumPy propose avec le format `npz`, de compresser facilement les
données :

```python
np.savez_compressed('test_compressed.npz', array=array, big_array=big_array)
loaded_npz = np.load('test_compressed.npz')
array_loaded_from_npz = loaded_npz['array']
big_array_loaded_from_npz = loaded_npz['big_array']
```

Comme pour `loadtxt` la function `load` se charge de décompresser
automatiquement.

## Conclusion

Voilà vous êtes maintenant capable de faire persister vos données numpy.

## Voir aussi

- [La doc de NumPy](https://numpy.org/doc/stable/reference/index.html)
- [Comment faire des requêtes HTTP en python avec requests](https://blog.jaaj.dev/2020/05/22/Comment-faire-des-requetes-http-en-python-avec-requests.html)
