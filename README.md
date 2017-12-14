# Aperture Data Studio SDK

The SDK provides a simple Java library to create your own custom steps and extend the capabilities of Aperture Data Studio.

This repo contains the SDK jar and a pre-configured Java project that uses Gradle to easily build your own custom step. Alternatively, you can add the SDK as a dependency to your own project by downloading the SDK jar from the `libs` folder.


## Generating a custom step with the sample project

The steps below show how to generate a compatible jar file using Gradle:

1. Clone the repo.
2. Open the project in your favourite IDE.
3. Create a new class within the MyCustomSteps module. For the output jar to work correctly it will need to be in the com.experian.aperture.datastudio.sdk.step.addons package - the template class is located there. We recommend that you base your class on one of the examples or use the template class.
4. Open the Gradle window in your IDE and run the MyCustomSteps build task. This will build the jar for the steps you have created.
5. Your new jar will be built to build/libs/MyCustomSteps.jar.

## Generating a custom step from a new or existing project 

If you don't wish to use Gradle, you'll need to configure your own java project to generate a compatible jar artifact:

1. Create a new java project or open an existing one.
2. Download the [sdk.jar](https://github.com/experiandataquality/aperture-data-studio-sdk/raw/master/libs/sdk.jar) file.
3. Create a libs folder and add in the sdk.jar as a library.
4. Create a new package - com.experian.aperture.datastudio.sdk.step.addons.
5. Create a new class in the package you just created.
4. Configure your project to output a jar file as an artifact, this will be done differently depending on your IDE.
 
## Adding a custom step to Aperture Data Studio

To make your custom step available in the Aperture Data Studio UI:

1. Copy your new jar into the addons folder in your Aperture Data Studio installation directory.
2. Restart the Aperture Data Studio service.
3. Test your new step by dragging it into the workflow like any other step.

## Examples

The project comes with an ExampleSteps module which, when built, will output the SDK examples jar. The example classes demonstrate some key functionality of the SDK along with providing a template class which can be used as a starting point for your own custom steps.

## Javadoc

You can view the Javadoc [here](https://experiandataquality.github.io/aperture-data-studio-sdk)

## Creating a custom step 

The sample project in the repository has a pre-configured Gradle build as well as having the SDK included and the correct package name configured. If you would like to start your own project then follow the instructions above.

With your project set up you can now create a new class. If you cloned the sample project you will have a MyCustomSteps module, inside that module you'll find the package com.experian.aperture.datastudio.sdk.step.addons. Create your new classes in this package so that they are correctly picked up by the Data Studio UI.

### Importing the SDK

In order to make use of the classes and methods, you'll need to import the SDK into your class. Add an import statement below the package name to import all the SDK classes and methods.
``` java
import com.experian.aperture.datastudio.sdk.step.*
```
Your new class should look something like this:

``` java
package com.experian.aperture.datastudio.sdk.step.addons;

import com.experian.aperture.datastudio.sdk.step.*;

public class DemoStep {
}
```
All the classes and methods will now be available to you.

### Configuring your step

The SDK has a `StepConfiguration` class. A custom step class should extend the `StepConfiguration` class. This will allow you to correctly configure your custom step and ensure it displays correctly in the UI.

You can create a new method in your class to set up your step. 

#### Adding step information

Each step needs some basic information to identify it in the Data Studio UI. You'll need to make sure your step has a name, description and icon:

``` java
package com.experian.aperture.datastudio.sdk.step.addons;

import com.experian.aperture.datastudio.sdk.step.*;

import java.util.Collections;
import java.util.Optional;
import java.util.Base64;

public class DemoStep extends StepConfiguration {

    public DemoStep() {
        // Basic step information
        setStepDefinitionName("DemoStep");
        setStepDefinitionDescription("Demonstrates a step");
        setStepDefinitionIcon("INFO");
    }
}
```

#### Adding step properties

Step properties represent step UI elements. These properties could include displaying information about the step, allowing the user to input something or selecting a column to manipulate. The property type is set using the `ofType` method. For example setting the property to be a column chooser would be done with the following code:
``` java
StepProperty arg1 = new StepProperty()
        .ofType(StepPropertyType.COLUMN_CHOOSER);
```

It it also recommended that you update the UI with some prompts and error icons to show the user that more interaction is required before the step will work correctly. You can do this using the `withStatusIndicator`, `withIconTypeSupplier` and `withArgTextSupplier` methods. The example below will show an error icon and display a couple of prompts if no data input is present and subsequently if no column is selected. If all is correct then the name of the column will be displayed.

``` java
StepProperty arg1 = new StepProperty()
        .ofType(StepPropertyType.COLUMN_CHOOSER)
        .withStatusIndicator(sp -> () -> sp.allowedValuesProvider != null)
        .withIconTypeSupplier(sp -> () -> sp.allowedValuesProvider == null ? "ERROR" : "OK")
        .withArgTextSupplier(sp -> () -> sp.allowedValuesProvider == null ? "Connect an input for columns" : (sp.getValue() == null ? "Select a column" : sp.getValue().toString()));
```
Most workflow steps will take in an input and then output something at the other end. To allow input and output you'll need to use `havingInputNode` and `havingOutputNode`. The final part of initial setup for a step property is to call `validateAndReturn` to perform the validation.

``` java
StepProperty arg1 = new StepProperty()
        .ofType(StepPropertyType.COLUMN_CHOOSER)
        .withStatusIndicator(sp -> () -> sp.allowedValuesProvider != null)
        .withIconTypeSupplier(sp -> () -> sp.allowedValuesProvider == null ? "ERROR" : "OK")
        .withArgTextSupplier(sp -> () -> sp.allowedValuesProvider == null ? "Connect an input for columns" : (sp.getValue() == null ? "Select a column" : sp.getValue().toString()))
        .havingInputNode(() -> "input0")
        .havingOutputNode(() -> "output0")
        .validateAndReturn();
```
The property is now ready to be added. The `setStepProperties` method takes a list. For a single property use a `SingletonList`. To add multiple properties, use `Arrays.asList`.

``` java
setStepProperties(Collections.singletonList(arg1));
```

Step output is where the main work is done, you'll need to define a new output class and set it using `setStepOutput`. This method takes a new `StepOutput` class:

``` java
setStepOutput(new DemoOutput());
```

### Configuring step output

Step output classes are configured by extending the `StepOutput` class.

```java
private class DemoOutput extends StepOutput {
}
```

First up you can set the name that appears when viewing the output data by overriding the getName method.

``` java
@Override
public String getName() { return "Demo step"; }
```
Next is the `initialise` method. This method initialises the view and so is where you would set up your output columns. You may want to add some columns or replace values in an existing column. You can use the `ColumnManager` class for this.

An example of using the column manager to add a column is below. Here the column manager clears the columns, gets the column selected by the user and adds a column next to it.


``` java
public void initialise() throws Exception {

    getColumnManager().clearColumns();

    String selectedColumnName = getArgument(0);
    if (selectedColumnName != null) {

        getColumnManager().setColumnsFromInput(getInput(0));

        StepColumn selectedColumn = getColumnManager().getColumnByName(selectedColumnName);
        if (selectedColumn != null) {
            int selectedColumnPosition = getColumnManager().getColumnPosition(selectedColumnName);

            getColumnManager().addColumnAt(this, selectedColumnName, "Base64 Encoded column", selectedColumnPosition);
        }
    }
}
```

### Using getValueAt
The `getValueAt` object is called for each cell when generating the view or executing the workflow. By default it will simply display the data as it is. If you override this, you can set the values in a specific column.
You'll see in the example below that the row and column are passed in. The example also shows getting the column selected by the user and using those values to set the values of another column.

``` java
@Override
public Object getValueAt(long row, int col) throws Exception {

    // get the user-defined column
    String selectedColumnName = getArgument(0);

    // get the column object from the first input
    Optional<StepColumn> inputColumn = null;
    if (selectedColumnName != null && !selectedColumnName.isEmpty()) {
        inputColumn = getInputColumn(0, selectedColumnName);
    }
    if (inputColumn.isPresent()) {
        // get the input column's value for the selected row
        String value = inputColumn.get().getValue(row).toString();
        // add text and return it
        return Base64.getEncoder().encodeToString(value.getBytes("utf-8"));
    } else {
        // if not found return an empty value. We could alternatively throw an error.
        return "";
    }
}
```

## Multi-threading

In order to improve performance, especially when calling a web service that may have slower response times, it is beneficial to to use multiple threads. The EmailValidate example step demonstrates how to make use of multi-threading within a custom step.

