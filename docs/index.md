---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---
<style>
.progress-bar {
background: linear-gradient(to right, red var(--scroll), transparent 0);
background-repeat: no-repeat;
width: 100%;
position: fixed;
top: 0;
left: 0;
height: 4px;
z-index: 1;
}
</style>

<!-- This is the bar which shows scroll percentage -->
<div class="progress-bar"></div>

<!-- Script used to generate --scroll variable with current scroll percentage value -->
<script>
var element = document.documentElement,
body = document.body,
scrollTop = 'scrollTop',
scrollHeight = 'scrollHeight',
progress = document.querySelector('.progress-bar'),
scroll;

document.addEventListener('scroll', function() {
scroll = (element[scrollTop]||body[scrollTop]) / ((element[scrollHeight]||body[scrollHeight]) - element.clientHeight) * 100;
progress.style.setProperty('--scroll', scroll + '%');
});
</script>


# Report on my investigation on the HDFS dataset

This page summarises my investigation on the dataset containing the digest of an HDFS
storage system log. The associated code, and this writeup, are available on
[github](https://github.com/musella/logmind_chall){:target="_blank"}.

I went through the following steps:
- basic data exploration, preprocessing and cleanup;
- data exploration and visualisation;
- data modelling.

I dedicated roughly 12 working hours to this investigation, which is to be considered as
the stub of a project.

## Executive summary

The data analyzed corresponds to a time **span** of roughly **two days**, and it contains **no
labels**, i.e. it is not know whether the data correspond to a period where the system was
behaving as expected.

The data exploration and analysis went therefore in two directions:
- **qualitative exploration** the patterns in the data;
- leveraging of **unspervised learning** to identify possible anomalous behaviour.

I created small **dashboard** to visualise the input data and the result of the data
model. The dashbord contains four rows of time series:
- the first three rows visualise the **event rates** grouped by template, sytem component and
  severity level;
- the forth row contains the results of an **anomaly score** predicted by a time-aware
  **auto-regressive machine learning model**.

All data are aggregated on the time scale of one minute. The model prediction are shown
only for the second  part of the dataset, which was not used for training.

For events above a "warning threshold", an analysis of the model output was performed to
**identify** the possible **sources of anomaly**. The result of such analysis are visualised by
the dashboard as the mouse hovers over the data points.

Given the limited size of the dataset, the model has **limited predictive power**. The events
that were identified most clearly as **anomalous** were related to some **data deletion** that
took place in the last part of the dataset. In particular the model marked as anomalous
the rate of the following kind of events
- `exception while serving <*> to <*>`
- `BLOCK* NameSystem.delete <*> is added to invalid set of <*>`
- `Deleting block <*> file <*>`

and similar instances.

Further details on the investigation process can be found below.

[![dashboard_thumb.png](/logmind_chall/assets/dashboard_thumb.png)](/logmind_chall/assets/dashboard.html){:target="_blank"}

## Preprocessing and cleanup

The data was split into two files:
- `hdfs_structured.csv`
- `hdfs_templates.csv`

The first file contains the acutal set of digested logs, while the second references a set
of event templates that were defined for the analysis of the data.

The data file contains the following columns:
`LineId,Date,Time,Pid,Level,Component,Content,EventId,EventTemplate`

The field `Content` contains the log message content, while `EventTemplate` references the category of event to which the message was associated.

Inspecting the actual event templates, I identified a few issues, that I

1. missing closed `[` parenthesis
```
1 <*> <*> <*> <*> java.io.InterruptedIOException: Interruped while waiting for IO on channel java.nio.channels.SocketChannel[connected <*> <*> <*> millis timeout left.
2 <*> <*> <*> <*> java.net.SocketTimeoutException: <*> millis timeout while waiting for channel to be ready for <*> ch : java.nio.channels.SocketChannel[connected <*> <*>
24 Exception in receiveBlock for block <*> java.io.InterruptedIOException: Interruped while waiting for IO on channel java.nio.channels.SocketChannel[connected <*> <*> <*> millis timeout left.
25 Exception in receiveBlock for block <*> java.net.SocketTimeoutException: <*> millis timeout while waiting for channel to be ready for write. ch : java.nio.channels.SocketChannel[connected <*> <*>
```
1. some template argument not recognised
```
26 PacketResponder <*> 1 Exception java.io.IOException: The stream is closed
27 PacketResponder <*> 1 Exception java.io.InterruptedIOException: Interruped while waiting for IO on channel java.nio.channels.SocketChannel[closed]. <*> millis timeout left.
43 writeBlock blk_1684134505299265593 received exception java.net.NoRouteToHostException: No route to host
```
1. different templates for what seems to be a special case of another template.
```
28 PacketResponder <*> <*> Exception <*>
29 PacketResponder <*> <*> Exception java.io.IOException: Broken pipe
```

During the preprocessing step, I fixed the second issue, while the other two will need to
be addressed at a later stage, depending in particular on the kind of application of these
data.

[Data preprocessing](https://github.com/musella/logmind_chall/blob/master/notebooks/cleanup_and_prepare.ipynb) included the follwoing steps:
1. *encoding of categorical data* (`EventTemplate`, `Level`, `Component`) as discrete
variables;
1. *extraction* of template arguments from the event content: the event content is thus
represented by the template id and the list of arguments associated to each template
(mostly data blocks and ip addresses);
1. digitisation of the event time stamp.


## Exploration and visualization

The dataset is composed by two kind of data:
- categorical data related to the template classification, system component and severity
level: this information is very homogeneous and its modelling is straightforward.
- unstructured data related to the template argument, which contain information on
the interaction between physical components of the system, and between the system and
external entities. The use of this data requires further processing to allow emergency of
the underlying dynamical graph structure.

Given the limited amount of time allocated for the investigation, I mostly concentrated on
the first kind, and in particular on the analysis of the rate of events in the different
categories.
For this, I first of all aggregated the data over periods of 1 minute, which I assumed to
be a good compromise between time resolution and sparsity of the data.


After the [aggregation
process](https://github.com/musella/logmind_chall/blob/master/notebooks/aggregate.ipynb),
the data is represented by 55 event rate columns, out of wich
- 44 are related to the event templates;
- 9 are related to the system components;
- the remaining 2 are related to the severity levels.

The total rate is clearly the same when summed over each of these three items. In the
dashobard the three items are plotted separately. Inspecting the corresponding plots, one
can identified several regular patterns, which we assume are related to the normal
behaviour of system.

## Data modelling

In the absence of any label on the state of the system, I tried to model the data
leveraging **unsupervised learning**, focusing in particular on auto-regressive models, that
are often used in anomaly-detection algorithms.

I trained two neural-network based models:
- a feed-forward **autoencoder** network working on a signle time slice;
- a **recurrent neural** network working on 30 time slices.


### Autoencoder

Autoencoders are neural network trained to reproduced their input, but structured in such
a way that they are forced to perform dimensionality reduction.

![autoencoder.png](/logmind_chall/assets/autoencoder.png){:width="50%"}

A way to apply them to anomaly detection algorithms is to train them on normal data and
then use the reconstruction error as an anomaly score. At training time, in fact, the
autoencoder learns efficient small dimensional representations for normal data, which do
not generalise well to anomalous samples.
Therefore, the autoencoder will perform poorly on anomalous data, giving rise to a large
reconstruction error.

The data was split into a training an testing sample with a roughly 80:20
proportion. Rather than using random splitting of the data, I took the test data to be the
last part of the dataset. This was done to work in a realistic condition where only
past data can be used for training.

I choose three encoding layers, with outputs of dimensions 32, 16 and 8. The output of the
decoding layer were instead 16, 32 and 55. I performed a few tests with network
regularisation and eventually settled on using dropout with a probability of 0.4.

For the loss function I test both mean squared error and mean absolute error (after input
standardisation), finding comparable results.

The figure below shows the distribution of the reconstruction error for the training and
test set.
![reconstruction_error.png](/logmind_chall/assets/reconstruction_error.png)

Based on this distribution, the model does signal anomalous events. In the following, I
nevertheless set an "anomaly warning threshold" at 200.

For the events above threshold, I inspected the model prediction to identify the possible
cause of the anomaly. In particular I looked at the difference between the rate in each of
the categories and the corresponding prediction. Sorting the categories by the absolute
deviation, one can identify the suspicious rates.

![causes.png](/logmind_chall/assets/causes.png){:width="60%"}

### RNN

In the second model I tried to include the time dimension in the analysis. This was done
exploiting a recurrent neural network, based on GRU units.

In particular, I trained a so called many-to-one model, i.e. a model mapping a sequence of
samples to a single instance. 

![autoencoder.png](/logmind_chall/assets/rnn.png){:width="50%"}

The network was trained on sequences made by 30 time slices to predict the even rate in
the 31st slice. Each input was fed into a dense layer reducing the dimensionality of each
time slice from 55 to 8. These 30 slices were then fed into 8 GRU units, and the final
output decoded with three layers of size 16, 32 and 55.

The RNN improves the mean reconstuction error by roughly 10% compared to the autoencoder
network (from 1.05 to 0.95 on the validation set). On the other hand, the two networks
give very similar reconstruction errors on the test set, especially for what concerns the
outliers. 

![reconstruction_error_scatter.png](/logmind_chall/assets/reconstruction_error_scatter.png)

While on the single time-slice the use of a recurrent network do not qualitative change
the accuracy of the model, the ability to exploit the temporal information, and to model
sequence of data samples make the recurrent approach worth pursuing further.

The training code is available
[here](https://github.com/musella/logmind_chall/blob/master/notebooks/train_autoencoder.ipynb) 
and
[here](https://github.com/musella/logmind_chall/blob/master/notebooks/train_rnn.ipynb).


## Outlook

The results described here are based on the preliminary analysis of a small
dataset. Further progress can be surely made by exploiting a larger dataset.

On the data modelling side several additional studies would be very useful to perform. In
particular, a general optimisation of the model hyper parameters may allow to improve
the performance. The recurrent network approach should be extended to analyse sequence of
data instead of individual time slices.

Also, the limited time did not allow to mine the unstructured data which can provide
additional, valuable information. A first step in this process would be the development of
an efficient embedding model to represent the different entities involved (data blocks, ip
addresses, execution exceptions, etc..).
In a second step one would model the interaction between the different entities using a
graphical model to mine valuable information to be used in the time-aware model.
