# Correlacion en los datos

_Este codigo es una continuación de [outliers code](./5_outliers_code.md)_.

Para esto vamos a hacer una matriz de correlación y ver los valores

```python
data_correlation = dataset.corr()
mask = np.zeros_like(data_correlation, dtype=np.bool)
mask[np.triu_indices_from(mask)] = True
f, ax = plt.subplots(figsize=(11, 9))
cmap = sns.diverging_palette(220, 10, as_cmap=True)
sns.heatmap(data_correlation, mask=mask, cmap=cmap,  center=0,
            square=True, linewidths=.5)
```



![png](./img/output_20_1.png)

```python
data_correlation['Class'].sort_values(ascending=False)
```


    Class                          1.000000
    Uniformity of Cell Shape       0.821891
    Uniformity of Cell Size        0.820801
    Bland Chromatin                0.758228
    Normal Nucleoli                0.718677
    Clump Thickness                0.714790
    Marginal Adhesion              0.706294
    Single Epithelial Cell Size    0.690958
    Mitoses                        0.423448
    Name: Class, dtype: float64


```python
dataset = dataset.drop(' Uniformity of Cell Shape',1)
```

Podemos ver que "Uniformity of Cell Shape" y "Uniformity of Cell Size" estan muy relacionadas entre si, por lo cual seria conveniente descartar alguno de los dos.

[CART ➡](./7_CART_code.md)