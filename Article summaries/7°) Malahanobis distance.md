# Malahanobis distance

We want to find a norm that allows us to calculate the similarity between two vectors from a distribution.
The Euclidean norm is not suitable because it is invariant with respect to direction, so a vector that is modified equally
on a random coordinate with small or large variance will remain the same distance from the new vector, whereas the distance from 
the distribution is greater in the case of modification on the small variance.

![mahalanobis3.png](mahalanobis3.png)

# Solution :

Reminder: 

In a probability distribution, variance gives us the bounds within which the data lies,
while covariance tells us the direction that this data is taking. For example in 2d : 

<p float="left">
  <img src="variance.png" width="500" />
  <img src="covariance.PNG" width="500" />
</p>


To construct a Malahanobis distance starting from our Euclidean distance, we must normalize the distances on 
the correlation axis and the perpendicular axis proportionally to the variances of the random variables.

![mahalanobis4.png](mahalanobis4.png)

To do this, we use the 
root of the covariance matrix (we take the root because we normalize with standard deviations and not variance). 
The covariance matrix gives us the normalizations in the canonical basis on the diagonal, 
and the other covariance terms allow us to recalculate this normalization in the
directions of correlation. For example, in two dimensions:

![matricedecovariance.PNG](matricedecovariance.PNG)

To illustrate how this matrix works, we can see how it transforms a vector. Since it is
symmetric, we can diagonalize it with rotation matrices.

![diagonalisation.PNG](diagonalisation.PNG)

When a vector is multiplied by this matrix,
it will be expressed in a new basis created by rotating the first one, then normalized on the 
axes of the new basis with the diagonal coefficients, then reversed to return to the original basis.
 We therefore normalize according to the desired axes: correlation axis and perpendicular axis.

In reality, in the calculation, we use the inverse of the root of the  covariance matrix because 
the covariance matrix increases the distances in the directions where the variance is large, which is the opposite 
of what we want. Ultimately,
what the inverse of the root of the  covariance matrix does is transforming the ellipse in the starting space into 
a circle
in the arrival space, and the Mahalanobis distance is the Euclidean distance calculated on this circle.

![distanceeuclidienne.PNG](distanceeuclidienne.PNG)
