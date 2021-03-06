# Data Management and Versioning <!-- omit in toc -->

- [Overview](#overview)
- [Datasets](#datasets)
- [Models](#models)
- [Plots](#plots)

## Overview

An artifact is any data that is produced and/or consumed by functions or jobs.

The artifacts are stored in the project and are divided to 3 main types:
1. **Datasets** — any data , such as tables and DataFrames.
2. **Plots** — images, figures, and plotlines.
3. **Models** — all trained models.

From the projects page, click on the **Artifacts** link to view all the artifacts stored in the project
<br><br>
<img src="_static/images/project-artifacts.png" alt="projects-artifacts" width="800"/>

You can search the artifacts based on time and labels.
In the Monitor view, you can view per artifact its location, the artifact type, labels, the producer of the artifact, the artifact owner, last update date.

Per each artifact you can view its content as well as download the artifact.

## Datasets

Storing datasets is important in order to have a record of the data that was used to train the model, as well as storing any processed data. MLRun comes with built-in support for DataFrame format, and can not just store the DataFrame, but also provide the user information regarding the data, such as statistics.

The simplest way to store a dataset is with the following code:

``` python
context.log_dataset(key='my_data', df=df)
```

Where `key` is the the name of the artifact and `df` is the DataFrame. By default, MLRun will store a short preview of 20 lines. You can change the number of lines by using the `preview` parameter and setting it to a different value.

MLRun will also calculate statistics on the DataFrame on all numeric fields. You can enable statistics regardless to the DataFrame size by setting the `stats` parameter to `True`.

## Models

An essential piece of artifact management and versioning is storing a model version. This allows the users to experiment with different models and compare their performance, without having to worry about losing their previous results.

The simplest way to store a model named `model` is with the following code:

``` python
from pickle import dumps
model_data = dumps(model)
context.log_model(key='my_model', body=model_data, model_file='my_model.pkl')
```

You can also store any related metrics by providing a dictionary in the `metrics` parameter, such as `metrics={'accuracy': 0.9}`. Furthermore, any additional data that you would like to store along with the model can be specified in the `extra_data` parameter. For example `extra_data={'confusion': confusion.target_path}`

A convenient utility method, `eval_model_v2`, which calculates mode metrics is available in `mlrun.utils`.

See example below for a simple model trained using scikit-learn (normally, you would send the data as input to the function). The last 2 lines evaluate the model and log the model.

``` python
from sklearn import linear_model
from sklearn import datasets
from sklearn.model_selection import train_test_split
from pickle import dumps

from mlrun.execution import MLClientCtx
from mlrun.mlutils import eval_model_v2

def train_iris(context: MLClientCtx):

    # Basic scikit-learn iris SVM model
    X, y = datasets.load_iris(return_X_y=True)
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42)
    model = linear_model.LogisticRegression(max_iter=10000)
    model.fit(X_train, y_train)
    
    # Evaluate model results and get the evaluation metrics
    eval_metrics = eval_model_v2(context, X_test, y_test, model)
    
    # Log model
    context.log_model("model",
                      body=dumps(model),
                      artifact_path=context.artifact_subpath("models"),
                      extra_data=eval_metrics, 
                      model_file="model.pkl",
                      metrics=context.results,
                      labels={"class": "sklearn.linear_model.LogisticRegression"})
```

Save the code above to `train_iris.py`. The following code loads the function and runs it as a job. See the [quick-start page](quick-start.html#mlrun-setup) to learn how to create the project and set the artifact path. 

``` python
from mlrun import code_to_function

gen_func = code_to_function(name=train_iris,
                            filename='train_iris.py',
                            handler=train_iris,
                            kind='job',
                            image='mlrun/ml-models')

train_iris_func = project.set_function(gen_func).apply(auto_mount())

train_iris = train_iris_func.run(name=train_iris,
                                    handler=train_iris,
                                    artifact_path=artifact_path)
```

You can now use `get_model` to read the model and run it. This function will get the model file, metadata, and extra data. The input can be either the path of the model, or the directory where the model resides. If you provide a directory, the function will search for the model file (by default it searches for .pkl files)

The following example gets the model from `models_path` and test data in `test_set` with the expected label provided as a column of the test data. The name of the column containing the expected label is provided in `label_column`. The example then retrieves the models, runs the model with the test data and updates the model with the metrics and results of the test data.

``` python
from pickle import load

from mlrun.execution import MLClientCtx
from mlrun.datastore import DataItem
from mlrun.artifacts import get_model, update_model
from mlrun.mlutils import eval_model_v2

def test_model(context: MLClientCtx,
               models_path: DataItem,
               test_set: DataItem,
               label_column: str):

    if models_path is None:
        models_path = context.artifact_subpath("models")
    xtest = test_set.as_df()
    ytest = xtest.pop(label_column)

    model_file, model_obj, _ = get_model(models_path)
    model = load(open(model_file, 'rb'))

    extra_data = eval_model_v2(context, xtest, ytest.values, model)
    update_model(model_artifact=model_obj, extra_data=extra_data, 
                 metrics=context.results, key_prefix='validation-')
```

To run the code, place the code above in `test_model.py` and use the following snippet. The model from the previous step is provided as the `models_path`:

``` python
from mlrun.platforms import auto_mount
gen_func = code_to_function(name=test_model,
                            filename='test_model.py',
                            handler=test_model,
                            kind='job',
                            image='mlrun/ml-models')

func = project.set_function(gen_func).apply(auto_mount())

run = func.run(name=test_model,
                handler=test_model,
                params={'label_column': 'label'},
                inputs={'models_path': train_iris.outputs['model'],
                        'test_set': 'http://iguazio-sample-data.s3.amazonaws.com/iris_dataset.csv'}),
                artifact_path=artifact_path)
```

## Plots

Storing plots is useful to visualize the data and to show any information regarding the model performance. For example, one can store scatter plots, histograms and cross-correlation of the data, and for the model store the ROC curve and confusion matrix.

For example, the following code creates a confusion matrix plot using [sklearn.metrics.plot_confusion_matrix](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.plot_confusion_matrix.html#sklearn.metrics.plot_confusion_matrix) and stores the plot in the artifact repository:

``` python
    cmd = metrics.plot_confusion_matrix(model, xtest, ytest, normalize='all', values_format='.2g', cmap=plt.cm.Blues)
    context.log_artifact(PlotArtifact('confusion-matrix', body=cmd.figure_), local_path='plots/confusion_matrix.html')
```

[Back to top](#top)
