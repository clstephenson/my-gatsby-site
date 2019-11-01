---
title: Generic AsyncTask for DAO Insert, Update, and Delete in Android
date: "2019-05-14T23:46:37.121Z"
template: "post"
draft: false
slug: "/posts/android-generic-asynctask-for-dao-insert-update-delete/"
category: "Android"
tags:
  - "Android"
  - "Java"
description: "While working on an Android application for a course, I found myself wanting to create a single AsyncTask class that could handle the create, update, and delete actions from repository classes. With a little exploring and learning, here is how I did it using generics and a listener interface."
---

A little background on the project and myself... This project was for a course in mobile application development using Android. It required an application to track your college terms, courses and assessments, among other things. I decided to use the Room library for persistence, as well as ViewModels and LiveData. I created base classes or interfaces for my models, DAOs, and repositories. This was my first taste of developing for Android, or any mobile platform, so understand that this is just how I approached a problem that I wanted to solve. I'm sure there are other ways to achieve the same goal. Also, I am still learning my way through Java, design patterns and proper coding techniques... not an expert by any means.

While working on this application, I wanted to create a generic `AsyncTask` object that could be called from various repository classes. I followed the tutorials to get my application set up with the room persistence library, but did not want to repeat the same code in each repository for the background tasks to perform basic insert, update and delete functions. Therefore, my goal here was to figure out how to create a generic class that extends `AsyncTask`. That class would be able to run those three database operations for all of my DAOs. I didn't need the task to handle select operations, since I was returning `LiveData` objects for that, and the background task is taken care of automatically.

Furthermore, I wanted the activity classes to be able to update the UI accordingly when the tasks completed. I did not want to update the UI views directly from the `onPostExecute` methods of the background tasks, so I needed to configure the activities to listen for updates from the task without the task knowing what was in the activities.

Now that the intro and disclaimer are out of the way, let's get to the code!

## 1. Listener Interface

First, create an interface with a single method as shown here in the `OnDataTaskResultListener`. The method here passes a single parameter containing a custom result object.

```java
// OnDataTaskResultListener.java
public interface OnDataTaskResultListener {
    void onNotifyDataChanged(DataTaskResult result);
}
```

## 2. Custom Task Result

Next, I created the custom result class called `DataTaskResult`. This is the object that will be returned by the background task, and passed to the user interface through the interface's method. An instance of this class will hold the action, result of the task, an exception (if needed), and any data that needs to be returned. The result is an enum value of `SUCCESS` or `FAIL`, and the action is another enum value specifying `INSERT`, `UPDATE` or `DELETE` operations.

```java
// DataTaskResult.java
public class DataTaskResult {
    private Result result;
    private DataTask.Action action;
    private Object data = null;
    private Exception exception = null;

    public enum Result {
        SUCCESS, FAIL
    }

    public DataTaskResult(DataTask.Action action, Result result) {
        this.action = action;
        this.result = result;
    }

    // getters and setters removed for brevity

    public boolean isSuccessful() {
        return result == Result.SUCCESS;
    }
}
```

## 3. Generic AsyncTask

Now comes the actual task. This class, called `DataTask`, extends `AsyncTask` and has a generic param. The generic param type `T` must be a subclass of `BaseModel`, which is just a simple class I used that contains an ID field. All of my entity model classes extend `BaseModel`. The constructor for `DataTask` takes four parameters.

1. **daoClass** is the actual class definition of the DAO related to the generic type `T` param. In my case, it must extend `BaseDao`. If you do not use a base class for your DAOs, then change this parameter to `Class<?> daoClass`. Here, my generic type `T` is a Note object, and I have a matching DAO class called `NoteDao` which extends my `BaseDao` class. The reason I added this parameter is because I needed to cast the `BaseDao` instance to the subclass's type. There's probably a better way, but this is how I was able to get things to work properly.

2. **dao** is an instance of the daoClass type. So, if this second parameter is an instance of `NoteDao` called `dao`, then the first parameter would be equivalent to `dao.getClass()`;

3. **action** is just that - the enum value of `INSERT`, `UPDATE`, or `DELETE`.

4. **listener** is an object that implements the `OnDataTaskResultListener` interface.

```java
// DataTask.java
public class DataTask<T extends BaseModel> extends AsyncTask<T, Void, DataTaskResult> {
    private final OnDataTaskResultListener listener;
    private final BaseDao asyncTaskDao;
    private final Action action;

    public enum Action {
        INSERT, UPDATE, DELETE
    }

    public DataTask(Class<? extends BaseDao> daoClass, BaseDao dao, Action action, OnDataTaskResultListener listener) {
        this.action = action;
        asyncTaskDao = daoClass.cast(dao);
        this.listener = listener;
    }
//...
```

The `doInBackground` method takes a param of the generic type `T`. So, when calling the `execute` method of this task, I pass in an instance of my entity model, such as a Note in my case. Since I only pass in a single object, I just take the first param, and then check the action given by the constructor. Based on the action value, call the relevant dao method. If any data should be returned, set the data field of the `DataTaskResult` object. I also wrapped the switch in a `try/catch` block. This way, I was able to add the exeption to the `DataTaskResult` object and set its result field to `FAIL`. Otherwise, the result was `SUCCESS`. We then return the `DataTaskResult` object when complete.

```java
// DataTask.java continued
    @Override
    protected DataTaskResult doInBackground(final T... params) {
        DataTaskResult dataTaskResult = new DataTaskResult(action, DataTaskResult.Result.FAIL);
        try {
            T object = params[0];
            switch (action) {
                case DELETE:
                    asyncTaskDao.delete(object);
                    break;
                case INSERT:
                    long insertId = asyncTaskDao.insert(object);
                    dataTaskResult.setData(insertId);
                    break;
                case UPDATE:
                    asyncTaskDao.update(object);
                    break;
            }
            dataTaskResult.setResult(DataTaskResult.Result.SUCCESS);
        } catch (Exception e) {
            Log.w(TAG, "doInBackground: " + e.getLocalizedMessage(), e);
            dataTaskResult.setException(e);
            dataTaskResult.setResult(DataTaskResult.Result.FAIL);
        } finally {
            return dataTaskResult;
        }
    }
//...
```

Finally, the `DataTaskResult` object is passed to the listener in the `onPostExecute` method by calling the listener's `onNotifyDataChanged` method. This is possible because listener implements the `OnDataTaskResultListener` interface.

```java
// DataTask.java continued
    @Override
    protected void onPostExecute(DataTaskResult taskResult) {
        listener.onNotifyDataChanged(taskResult);
    }
}
```

## 4. Repository Executes the Task

Here is a simplified version of one of my repository classes that uses the `DataTask` class. There are a few other methods that I've removed since they're not important for this example. The main parts for our purposes here are the methods for `setOnDataTaskResultListener`, `delete`, `update`, and `insert`. The three database methods create a new instance of a `DataTask<Note>`, passing the dao class information, action, and registered listener. The actual note object to be affected is passed through the `execute` method.

```java
// NoteRepository.java
public class NoteRepository {
    private final NoteDao noteDao;
    private OnDataTaskResultListener onDataTaskResultListener;

    public NoteRepository(Application application) {
        AppDatabase db = AppDatabase.getDatabase(application);
        noteDao = db.noteDao();
    }

    public void delete(Note note) {
        new DataTask<Note>(noteDao.getClass(), noteDao, DataTask.Action.DELETE, onDataTaskResultListener).execute(note);
    }

    public void update(Note note) {
        new DataTask<Note>(noteDao.getClass(), noteDao, DataTask.Action.UPDATE, onDataTaskResultListener).execute(note);
    }

    public void insert(Note note) {
        new DataTask<Note>(noteDao.getClass(), noteDao, DataTask.Action.INSERT, onDataTaskResultListener).execute(note);
    }

    public void setOnDataTaskResultListener(OnDataTaskResultListener listener) {
        this.onDataTaskResultListener = listener;
    }
}
```

## 5. Activity Initiates Task and Listens for Result

Finally, we've come to the UI activity class. The activity should implement `OnDataTaskResultListener`.

```java
// MyActivity.java
public class MyActivity extends AppCompatActivity implements OnDataTaskResultListener {
```

In the `onCreate` method, or somewhere in the activity, get an instance of the repository and register itself as a listener. As a side note, you do not have to deal directly with the repository here. In my case, I actually used a `ViewModel` class, which contained wrappers for the repository functions. I'm just trying to simplify this example by removing the extra layer.

```java
// MyActivity.java
repository.setDataTaskResultListener(this);
```

Because, the activity implements the `OnDataTaskResultListener`, it must also override `onNotifyDataChanged`. This is where you can check the result and perform your UI updates.

```java
// MyActivity.java
@Override
public void onNotifyDataChanged(DataTaskResult result) {
    switch (result.getAction()) {
        case INSERT:
            if (result.isSuccessful()) {
                // do something if insert was successful
            } else {
                // do something if insert failed
            }
            break;
        case DELETE:
            if (result.isSuccessful()) {
                // do something if delete was successful
            } else {
                // do something if delete failed
            }
            break;
        case UPDATE:
            if (result.isSuccessful()) {
                // do something if update was successful
            } else {
                // do something if update failed
            }
            break;
    }
}
```
