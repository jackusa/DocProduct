# Doc Product: Medical Q&A with Deep Language Models

<p align="center">
  <img src="https://challengepost-s3-challengepost.netdna-ssl.com/photos/production/software_photos/000/806/964/datas/gallery.jpg">
</p>

## Quality medical information is valuable to everyone, but it's not always readily available. Doc Product aims to fix that.

Whether you've hit your head and are unsure if you need to see a doctor, caught a bad bug halfway up the Himalayas with no idea how to treat it, or made a pact with the ancient spaghetti gods to never accept healthcare from human doctors, *Doc Product* has you covered with up-to-date information and unique AI-generated advice to address your medical concerns.

We wanted to explore how well state-of-the-art natural language processing models like [BERT](https://arxiv.org/abs/1810.04805) and [GPT-2](https://openai.com/blog/better-language-models/) could respond to medical questions by retrieving and conditioning on relevant medical data, and this is the result.

## How we built it

As a group of friends with diverse backgrounds ranging from broke undergrads to data scientists to top-tier NLP researchers, we drew inspiration for our design from various different areas of machine learning. By combining the power of [attention transformer architectures](https://arxiv.org/abs/1706.03762), [latent vector search](https://github.com/facebookresearch/faiss), and [generative pre-training](https://d4mucfpksywv.cloudfront.net/better-language-models/language_models_are_unsupervised_multitask_learners.pdf), we were able to come up with a novel solution to a difficult problem that at first seemed like a herculean task.

<div style="text-align:center"><img src="https://i.imgur.com/wzWt039.png" /></div>

- 700,000 medical questions and answers scraped from Reddit, HealthTap, WebMD, and several other sites
- Fine-tuned TF 2.0 [Bidirectional Transformer](https://arxiv.org/abs/1810.04805) with [pre-trained BioBERT weights](https://arxiv.org/abs/1901.08746) for extracting representations from text
- Fine-tuned TF 2.0 [Generative Pre-training Transformer V2](https://d4mucfpksywv.cloudfront.net/better-language-models/language_models_are_unsupervised_multitask_learners.pdf) with OpenAI's GPT2-117M parameters for generating answers to new questions.
- Network heads for mapping question and answer embeddings to metric space, made with a Keras.Model feedforward network
- Over a terabyte of TFRECORDS, CSV, and CKPT data

## The challenges we faced

Our project was wrought with too many challenges to count, from having too much data to store on drive, to re-implementing the entirety of BERT in TensorFlow 2.0, to coordinating with teammates from across the planet, to waiting out astronomically long preprocessing and training times. However, what's important is that we faced those challenges as a team, and with each other's support and late-night pep talks over Google Hangouts, we rose to those challenges and overcame them together.

## What it does

Our BERT has been trained to encode medical questions and medical information. A user can type in a medical question, and our model will retrieve the most relevant medical information to that question.

###Data

We created datasets from several medical question and answering forums. The forums are WebMD, HealthTap, eHealthForums, iClinic, Question Doctors, and Reddit.com/r/AskDocs

###Architecture 

The architecture consists of a fine-tuned bioBert (same for both questions and answers) to convert text input to an embedding representation. The embedding is then input into a FCNN (a different one for the questions and answers) to develop an embedding which is used for similarity lookup. The top similar questions and answers are then used by GPT-2 to generate an answer. The full architecture is shown below. 

Lets take a look at the first half of the diagram above above in more detail, the training of the BERT and the FCNNs. A detailed figure of this part is shown below

![DoctorBert](https://i.imgur.com/IRCyKIL.jpg?1)

During training, we take a batch of medical questions and their corresponding medical answers, and convert them to bioBert embeddings. The same Bert weights are used for both the questions and answers. 

![DoctorBert](https://i.imgur.com/Lpjjcvk.jpg)

These embeddings are then inputted into a FCNN layer. There are separate FCNN layers for both the question and answer embeddings. 

![DoctorBert](https://i.imgur.com/6HwikW2.jpg)

Now here's where things get a little tricky. Usually embedding similarity training involves negative samples, like how word2vec uses NCE loss. However, we can not use NCE loss in our case since the embeddings are generated during each step, and the weights change during each training step. 

So instead of NCE loss, what we did was compute the dot product for every combination of the question and answer embeddings within our batch. This is shown in the figure below

![DoctorBert](https://i.imgur.com/KOyiCJU.jpg)

Then, a softmax is taken across the rows; for each question, all of it's answer combinations are softmaxed. 

![DoctorBert](https://i.imgur.com/X6N84Gd.jpg)

Finally, the loss used is cross entropy loss. The softmaxed matrix is compared to a ground truth matrix; the correct combinations of questions and answers are labeled with a '1', and all the other combinations are labeled with a '0'. 

## Challenges we ran into

### Data Gathering and Wrangling

### Converting Modules to be TF2.0 compatible

### Combining Models Built in TF 1.X and TF 2.0

The embedding models are built in TF 2.0 which utilizes the flexibility of eager execution of TF 2.0. However, GPT2 model that we use are are built in TF 1.X. Luckily, we can train two models separately. While inference, we need to maintain disable eager execution with tf.compat.v1.disable_eager_execution and maintain two separate sessions. We also need to take care of the GPU memory of two sessions to avoid OOM.

## Accomplishments that we're proud of

### Robust Model with Careful Loss and Architecture Design

One obvious approach to retrieve answers based on user’s questions is that we use a powerful encoder(BERT) to encode input questions and questions in our database and do a similarity search. There is no training involves and the performance of this approach totally rely on the encoder. Instead, we use separate Feed-forward networks for questions and answers and calculate cosine similarity between them. Inspired by the negative sampling of word2vec paper, we treat other answers in the same batch as negative samples and calculate cross entropy loss. This approach makes the questions embeddings and answers embeddings in one pair as close as possible in terms of cosine similarity. It turns out that this approach yields more robust results than doing similarity search directly using BERT embedding vector.

### High-performance Input Pipeline

The preprocessing of BERT is complicated and we totally have around 333K QA pairs and over 30 million tokens. Considering shuffle is very important in our training, we need the shuffle buffer sufficiently large to properly train our model. It took over 10 minutes to preprocess data before starting to train model in each epoch. So we used the tf.data and TFRecords to build a high-performance input pipeline. After the optimization, it only took around 20 seconds to start training and no GPU idle time. 

Another problem with BERT preprocessing is that it pads all data to a fixed length. Therefore, for short sequences, a lot of computation and GPU memory are wasted. This is very important especially with big models like BERT. So we rewrite the BERT preprocessing code and make use of [tf.data.experimental.bucket_by_sequence_length](https://www.tensorflow.org/versions/r2.0/api_docs/python/tf/data/experimental/bucket_by_sequence_length) to bucket sequences with different lengths and dynamically padding sequences. By doing this, we achieved a longer max sequence length and faster training.

### Imperative BERT Model

After some modification, the Keras-Bert is able to run in tf 2.0 environment. However, when we try to use the Keras-Bert as a sub-model in our embedding models, we found the following two problems.
- It uses the functional API. Functional API is very flexible, however, it’s still symbolic. That means even though eager execution is enabled, we still cannot use the traditional python debugging method at run time. In order to fully utilize the power of eager execution, we need to build the model using tf.keras.Model
- We are not directly using the input layer of Keras-Bert and ran into this [issue](https://github.com/tensorflow/tensorflow/issues/27543). It’s not easy to avoid this bug without changing our input pipeline.

As a result, we decided to re-implement an imperative version of BERT. We used some components of Keras-Bert(Multihead Attention, Checkpoint weight loading, etc) and write the call method of Bert. Our implementation is easier to debug and compatible with both flexible eager mode and high-performance static graph mode.

### Answer Generation with Auxiliary Inputs

Users may experience multiple symptoms in various condition, which makes the perfect answer might be a combination of multiple answers. To tackle that, we make use of the powerful GPT2 model and feed the model the questions from users along with Top K auxiliary answers that we retrieved from our data. The GPT2 model will be based on the question and the Top K answers and generate a better answer. To properly train the GPT2 model, we create the training data as following: we take every question in our dataset, do a similarity search to obtain top K+1 answer, use the original answer as target and other answers as auxiliary inputs. By doing this we get the same amount of GPT2 training data as the embedding model training data.

## What we learned

Bert is fantastic for encoding medical questions and answers, and developing robust vector representations of those questions/answers. 

We trained a fine-tuned version of our model which was initialized with Naver's bioBert. We also trained a version where the bioBert weights were frozen, and only trained the two FCNNs for the questions and answers. While we expected the fine-tuned version to work well, we were surprised at how robust later was. This suggests that bioBert has innate capabilities in being able to encode the means of medical questions and answers. 

## What's next for Information Retrieval w/BERT for Medical Question Answering 

Explore if there's any practical use of this project outside of research/exploratory purposes. A model like this should not be used in the public for obtaining medical information. But perhaps it can be used by trained/licenced medical professionals to gather information for vetting. 


Explore applying the same method to other domains (ie history information retrieval, engineering information retrieval, etc.).

Explore how the recently released sciBert (from Allen AI) compares against Naver's bioBert. 

## DISCLAIMER

The recommendations of an open-source AI application is not a substitute for professional medical care. If your condition is worsening, please go to your primary care provider. If you are having an emergency, please go to the nearest hospital or call your country's emergency number.

The purpose of this project is to explore the capabilities of deep learning language models. Although the application will help find publicly available medical advice, you are following it AT YOUR OWN RISK.

![DoctorBert](https://i.ytimg.com/vi/nPemP-Q0Xn8/hqdefault.jpg)

#Thanks

Special thanks to Llion Jones whose insights and guidance had a significant impact in the direction and progress of our project
