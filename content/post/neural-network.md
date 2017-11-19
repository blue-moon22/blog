---
title: Interview-Proof Deep Neural Network
tags: ["neural network", "machine learning", "deep learning"]
date: 2017-11-19
---

Recently, I had an interview for a data science job and I was asked to explain a neural network (which I responded with a quite hand-wavey answer). I haven't posted in a while so thought this would be a good opportunity to go through a simple deep neural network in R that I did a while back. This may be useful for those of you who need to prep up on those technical interview questions.

```
#### Deep Neural Network (one hidden layer, with Rectified Linear Unit (ReLU), L2 regularisation and Softmax loss function)

# Train DNN (for one hidden layer)
train.dnn <- function(predictorcols, labelcol, traindata = data,
                      hidden = 6, maxiterations = 2000, abstol=1e-2, 
                      learningrate= 1e-2, regularisation = 1e-3) 
{
  # Extract predictors and label
  predictors <- as.matrix(traindata[,predictorcols])
  label <- traindata[,labelcol]
  
  # Number of observations
  num_obs <- nrow(traindata)

  # create index for both row and col
  label_index <- cbind(1:num_obs, label)
  
  # number of input predictors
  predictor_num <- ncol(predictors)
  
  # number of categories for classification
  num_unique_labels <- length(unique(label))
  
  # initialise weights and biases
  W1 <- 0.01*matrix(rnorm(predictor_num*hidden, sd=0.5), nrow=predictor_num, ncol=hidden)
  b1 <- matrix(0, nrow=1, ncol=hidden)
  
  W2 <- 0.01*matrix(rnorm(predictor_num*hidden, sd=0.5), nrow=hidden, ncol=num_unique_labels)
  b2 <- matrix(0, nrow=1, ncol=num_unique_labels)
  
  # Training the network
  i <- 1
  while(i < maxiterations || loss < abstol) {

    #### Feed Forward

    # Matrix multiplication of predictors and 1st weights, add 1st biases
    hidden_layer <- sweep(predictors %*% W1, 2, b1, '+')
    # ReLU Activation Model: return value if greater than 0, otherwise return 0
    hidden_layer <- pmax(hidden_layer, 0)
    # Matrix multiplication of hidden layer and 2nd weights, add 2nd biases
    score <- sweep(hidden_layer %*% W2, 2, b2, '+')
    
    # Softmax Loss Function: creates a probability distribution
    score_exp <- exp(score)
    probs <- sweep(score_exp, 1, rowSums(score_exp), '/')
    
    # Compute the data loss
    true_logprobs <- -log(probs[label_index])
    data_loss <- sum(true_logprobs)/num_obs
    # L2 Regularisation Loss to prevent overfitting
    reg_loss <- 0.5*regularisation* (sum(W1*W1) + sum(W2*W2))
    # Total loss
    loss <- data_loss + reg_loss

    #### Backpropagation to minimise the loss
    
    # Get gradient of scores
    dscores <- probs
    dscores[label_index] <- dscores[label_index] - 1
    dscores <- dscores/num_obs
    
    # Backpropagate into weights and biases
    dW2 <- t(hidden_layer) %*% dscores
    db2 <- colSums(dscores)
    
    dhidden <- dscores %*% t(W2)
    dhidden[hidden_layer <= 0] <- 0
    
    dW1 <- t(predictors) %*% dhidden
    db1 <- colSums(dhidden)
    
    # Update weights and biases
    dW2 <- dW2 + regularisation*W2
    dW1 <- dW1 + regularisation*W1
    
    W1 <- W1- learningrate*dW1
    b1 <- b1 - learningrate*db1
    
    W2 <- W2-learningrate*dW2
    b2 <- b2 - learningrate*db2
    
    i <- i + 1
  }
  
  #### Return the final model
  
  model <- list(predictor_num = predictor_num, hidden = hidden, 
                num_unique_labels = num_unique_labels,
                W1 = W1, b1 = b1, W2 = W2, b2 = b2)
  return(model)
}

# Prediction function
predict.dnn <- function(model, data) {
  data <- as.matrix(data) # Change to matrix
  
  # Feed Forward
  hidden_layer <- sweep(data %*% model$W1 , 2, model$b1, '+')
  
  # ReLU
  hidden_layer <- pmax(hidden_layer, 0)
  score <- sweep(hidden_layer %*% model$W2, 2, model$b2, '+')
  
  # Softmax Loss Function
  score_exp <- exp(score)
  probs <- sweep(score_exp, 1, rowSums(score_exp), '/')
  
  # Select max possibility
  labels_pred <- max.col(probs)
  return(labels_pred)
}

# Read iris data
data(iris)
iris <- data.frame(iris, stringsAsFactors = F)

# Make the example reproducible
set.seed(123)

# Split into train and test
samp <- c(sample(1:50,25), sample(51:100,25), sample(101:150,25))

# Train DNN
trained_model <- train.dnn(predictorcols=1:4, labelcol=5, traindata=iris[samp,], 
                           hidden=6, maxiterations=2000)

# Predict from model
predicted_labels <- predict.dnn(trained_model, iris[-samp, -5])

# Continguency table
cont_table <- table(iris[-samp,5], predicted_labels)
cont_table

# Accuracy
sum(diag(cont_table))/sum(cont_table)
```