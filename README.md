# PyQt5-reorderable-list-model
**Example of reorderable via drag-n-drop list model in PyQt5**

## What's this

This small script is a demo of how one can implement a model within Qt's model-view framework which items can be reordered via drag-n-drop.

It is a continuation of work started in the answer to [this](http://stackoverflow.com/q/40250771/1217285) question on Stack Overflow. Now this demo uses PyQt5. It was tested with Python 3.4.3.

## How it works

As long as Qt's model-view framework's drag-n-drop requires the serialization of some data about the items being repositioned, the reordering via drag-n-drop can be implemented in the following way:

 * On drag we serialize some info about the dragged items
 * On drop we:
   * deserialize this info back
   * use this info to locate each dropped item's original position in the model - we need it to determine where to put the dropped item - before the item onto which we dropped or after it; the former option is useful if we drop onto the very first item and the latter option is useful in all other cases
   * insert the dropped item into the appropriate position within the model
   * if our model properly implements the `removeRows` method **and** the view's [dragDropOverwriteMode](http://doc.qt.io/qt-5/qabstractitemview.html#dragDropOverwriteMode-prop) property is set to `false` (as it is by default in `QListView` and `QTreeView` but not in `QTableView`), Qt would call it automatically after `dropMimeData` method if the latter one returns `true` thus confirming the drop was processed successfully.

From the model's perspective that wraps up the whole drag-n-drop story. However, there's one more important aspect: the selection. The selection in Qt's model-view framework belongs to the view part and thus the model knows nothing about the selection and how to update it properly. So we need to tinker with the view part as well if we want it to behave nicely.

Perhaps the simplest and most intuitive behaviour would be to have the dropped items selected and one of them to be set as the current item (`selected` and `current` are two different concepts within [QItemSelectionModel](http://doc.qt.io/qt-5/qitemselectionmodel.html)) after the reordering is finished. I haven't found a way to achieve this with standard `QItemSelectionModel` so I implemented a custom subclass of it and added a couple of things to the model to provide some useful info to the selection model:
 * The model now stores the list of last dropped items updated near the end of `dropMimeData` method
 * The model also sets the boolean flag in the end of `dropMimeData` method thus indicating that the next call to `removeRows` should be treated as the call performed by Qt automatically which indicates the end of drag-n-drop events sequence. So in the end of `removeRows` the model checks this flag and if it's set, it drops the flag back and emits a custom signal `dragDropFinished` which intends to informs whoever is listening about the fact that the whole drag-n-drop sequence is finished from the model's perspective.
 * The selection model connects a special slot to this model's signal in which it fetches the list of last dropped items from the model, figures out which rows these items now correspond to (using another custom method within the model - `rowForItem`) and selects these rows. It also sets the last of these rows as the current one.
