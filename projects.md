---
layout: default
---

# Independent projects - Machine Learning

During the last year of my doctoral studies, during my spare time, I have been exploring data and ways to analyze and model them. Some of the projects I worked on are the following:

1. I analyzed [histology tiles of Colorectal cancer patients](https://github.com/arunravishankar/Colorectal_Cancer_Histology/blob/master/Colorectal_Cancer_Images.ipynb) - used clustering algorithms to understand the data better and eventually built a Convolutional Neural Network model to predict the type of tissue of a given image. I used this as a testing ground to understand some of the basics of clustering, CNNs and also to analyze image data. 

2. I did [this](https://github.com/arunravishankar/Inferring-a-3D-line-from-2D-/blob/master/3DLine_from_2DPoints.pdf) as a part of the course 'Introduction to Machine Learning' I took in the Fall semester of 2019. In this Computer vision problem, given a set of points normally distributed about a line in 3 dimensions, I used a Metropolis-Hastings (Monte-Carlo Markov Chain) sampler to obtain the posterior distribution of the end points of the line segment about which these given points were distributed about in 3 dimensions given these points as viewed by a camera (2 Dimensions). I wrote the MH sampler from scratch in python and the relevant plots were made. This was repeated for the same points as viewed from a different camera (different angle). Given both these datasets, using the likelihood from both these distributions, I inferred the line segment with more confidence.

3. In order to get familiar with classification initially, I built and compared [classification models](https://github.com/arunravishankar/Cervical_cancer_risk_classification/blob/master/Cervical_cancer.ipynb) to predict a patient's risk to being diagnosed with Cervical Cancer.

4. I'm currently building regression models to predict drug response in cancerous tissues based on genomic data from the Genomic Data Commons Data Portal of the NIH. I am comparing different methods of feature selection on a high dimensional feature space to balance interpretability and accuracy of the supervised learning model in order to identify the biological pathway causing the cancer. 
