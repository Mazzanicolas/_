# Naive Bayes

### Naive Bayes Consideraciones

✔ Naive Bayes es para resolver problemas de clasificación.

* Nuestro problema es de clasificacion, predecir alguna de las categorias, 2 o 4.

✔ Naive Bayes asume que las label de salidas son categoricas _(puede tomar entras numericas pero es otro caso)_.

* En este caso nuestra salida es categorica 2 y 4, que son nuestras clases.

### Proceso en RapidMiner

__Seed = 2018__

1- Agregamos el dataset en un proceso nuevo con el modulo `Retrive`.

2- Eliminamos los atributos inecesarios con un modulo de `Select Attributes`, en este caso vamos a eliminar la id.

3- Como vimos en [Missing Values](./), este dataset contiene valores faltantes en el atributo **Bare Nuclei**. Vamos a removerlos con el modulo `Filter Examples`.

4- En los proximos pasos vamos a utlizar el modulo de `Performace (Classification)` el cual requiere que las _labels_ sean de tipo _polynomial_. Actualmente nuestras variables de salida son 2 y 4, para cumplir con este requsito vamos a utilizar el modulo `Numerical to Polynominal` sobre la variable **Class**.

5- Los valores del atributo **Bare Nuclei** estan siendo considerados como _polynomial_ vamos a utilizar el modulo de `Nominal to Numerical` para convertirlo en numeros.

El paso 4 y 5 los englobamos en un `Subprocess`.

6- Indicamos que el atributo **Class** va a ser nuestra _label_ a predecir con el modulo `Set Role`.

7- Para evaluar que tan buneo es nuestro modelo vamos a utilizar validacion cruza tambien conocido como _cross validation_, vamos a utilizar el estandar de 10 folds.

* Dentro del cross validation:
  
  7.1- En el lado izquierdo _(training)_ agregamos el modulo `Naive Bayes`.

  7.2- En el lado derecho _(testing)_ agregamos el modulo de `Apply model` conectado a `Performance (Classification)`.


### Process

![](./img/11_knn_process_3.PNG)

![](./img/11_knn_process_1.PNG)

### Cross Validation

![](./img/12_naive_bayes_process_2.PNG)

## Experimentos

El modulo de `Naive Bayes` solo cuenta con un hyperparametro, **laplace correction**.

| Laplace Correction   | Accuracy         | 2 Recall | 4 Recall |
|----------------------| ---------------- |--------- | -------- |
|**✔**                   | **96.34% +/- 1.88%**  | **95.27%**  | **98.33%**  |
| ✘                   | 96.34% +/- 1.88%  | 95.27%   | 98.33%  |

## Resultados

Como era de epserarse, tener activado o desactivado la correcion de laplace no cambia los resutados ya solo surge efecto cuando no tenemos ninguna instancia de alguna de las categorias.