2024-11-01 17:48

__Status__: #Infante 
__Tags__: #PyE #LeyDeGrandesNúmeros #Métodos #MétodosDeEstimación #MáximaVerosimilitud #FaMAF 
# Ley de Grandes Números y Métodos de ...
##### Volver a [[Probabilidad y Estadística parte 2]]

#### Bullet points para ir ampliando:
TODO list
Basada en el [cronograma](https://docs.google.com/document/d/1j65bz3ai-FK4LxX7xpiIAjVU0zfqKKxw/edit) de las clases del 3 y 15 de octubre.
- [ ]  Ley de grandes números, consistencia de un estimador.
- [ ] Teorema central del límite.
- [ ] Aproximación normal a la  binomial.
- [ ] Métodos de estimación puntual.
- [ ] Método de los momentos.
- [ ] Estimador para $\boldsymbol{\sigma}$ . #WIP
- [x] Método de máxima verosimilitud.
- [ ] Propiedad de invarianza. #WIP
- [ ] Propiedades asintóticas del estimador de MV. 




1. ***Estimador para*** $\boldsymbol{\sigma}$  #completar
Para distribución normal, para máxima verosimilitud se usa $n$ en lugar de $n - 1$:

$$
\hat{\sigma} = S = \sqrt{\frac{1}{n - 1} \sum_{i=1}^{n} (X_i - \bar{X})^2}
$$
Para distribución exponencial

$$
\hat{\sigma} = \bar{X}
$$
Para muestra general (sin distribución específica)

$$
\hat{\sigma} = \sqrt{\frac{1}{n - 1} \sum_{i=1}^{n} (X_i - \bar{X})^2}
$$


2. ***Estimador de máxima verosimilitud***
Supongamos que se quiere estimar un sólo parámetro $\theta$ , el estimador  $\hat{\theta}$ de máxima verosimilitud es el valor de $\theta$ que maximiza la verosimilitud de $L({\theta})$, esto es:

$$
\hat{\theta} = \arg \max_{\theta} \prod_{i=1}^{n} p(x_i \mid \theta)
$$

Ese **estadístico** también realiza el máximo de la función de log-verosimilitud $\ell(\theta) \equiv \ln p(\mathbf{x}_1, \dots, \mathbf{x}_n \mid \theta)$, esto es:

$$
\hat{\theta} = \arg \max_{\theta} \sum_{i=1}^{n} \ln p(\mathbf{x}_i \mid \theta)
$$


Generalizando la verosimilitud como una función de los $k$ parámetros desconocidos $\theta=(\theta_1, \dots, \theta_k)$; el estimador vectorial de máxima verosimilitud es el vector $\hat{\theta} = (\hat{\theta}_1, \dots, \hat{\theta}_k)$ que maximiza la $\ell(\theta) \equiv \ln p(\mathbf{x} \mid \theta)$

$$
\hat{\theta} = \arg \max_{\theta} \sum_{i=1}^{n} \ln p(\mathbf{x}_i \mid \theta)
$$

En la mayoría de los casos, esto se reduce a resolver la ecuación

$$
\nabla_{\theta} \mathcal{L}(\theta) = \sum_{i=1}^{n} \nabla_{\theta} \ln p(\mathbf{x}_i \mid \theta) = 0
$$
donde el operador gradiente es

$$
\nabla_{\theta} \equiv \left[ \frac{\partial}{\partial \theta_1}, \dots, \frac{\partial}{\partial \theta_k} \right]^T
$$

3. ***Propiedad de invarianza del estimador por MV*** #completar 

$$
P_{95} = \mu + z_{0.95} \cdot \sigma
$$

$$
\hat{P}_{95} = \hat{\mu} + z_{0.95} \cdot \hat{\sigma}
$$

## Referencias

- [[Probabilidad y estadística material]]

- **Clases referenciadas:**
	-  [Clase 3 de octubre](https://docs.google.com/presentation/d/1L3KN8-Ey8zvhiVCgiMA5x_f0trYGl2WJJTkkyuys_2k/edit#slide=id.p)

	-  [Clase 15 de octubre](https://docs.google.com/presentation/d/1dYOHOgDO-ZGa20zGizzSKM-kB-TjKKjtiJHDXEMQJrI/edit#slide=id.p)




