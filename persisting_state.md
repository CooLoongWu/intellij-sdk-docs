---
title: Persisting State of Components
---

<!--INITIAL_SOURCE https://confluence.jetbrains.com/display/IDEADEV/Persisting+State+of+Components-->

# {{ page.title }}

IntelliJ IDEA provides an API that allows components or services to persist their state between restarts of IntelliJ IDEA. 
You can use either a simple API to persist a few values, or persist the state of more complicated components using the 
[PersistentStateComponent](http://git.jetbrains.org/?p=idea/community.git;a=blob;f=platform/core-api/src/com/intellij/openapi/components/PersistentStateComponent.java;hb=HEAD) 
interface.

## Using PropertiesComponent for Simple non-roamable Persistence

If the only thing that your plugin needs to persist is a few simple values, the easiest way to do so is to use the ```com.intellij.ide.util.PropertiesComponent``` service. It can be used for saving both application-level values and project-level values (stored in the workspace file). Roaming is disabled for PropertiesComponent, so, use it only for temporary non-roamable properties.

Use the ```PropertiesComponent.getInstance()``` method for storing application-level values, and the ```PropertiesComponent.getInstance(Project)``` method for storing project-level values.
Since all plugins share the same namespace, it is highly recommended to prefix key names (e.g. using your plugin ID).

## Using PersistentStateComponent

The ```com.intellij.openapi.components.PersistentStateComponent``` interface gives you the most flexibility for defining the values to be persisted, their format and storage location. In order to use it, you should mark a service or a component as implementing the ```PersistentStateComponent``` interface, define the state class, and specify the storage location using the ```@com.intellij.openapi.components.State``` annotation.

Note that instances of extensions cannot persist their state by implementing ```PersistentStateComponent```. If your extension needs to have persistent state, you need to define a separate service responsible for managing that state.

### Implementing the PersistentStateComponent Interface

The implementation of ```PersistentStateComponent``` needs to be parameterized with the type of the state class. The state class can either be a separate JavaBean class, or the class implementing ```PersistentStateComponent``` itself.

In the former case, the instance of the state class is typically stored as a field in the ```PersistentStateComponent``` class:

```java
class MyService implements PersistentStateComponent<MyService.State> {
  class State {
    public String value;
  }
  State myState;
  public State getState() {
    return myState;
  }
  public void loadState(State state) {
    myState = state;
  }
}
```

In the latter case, you can use the following pattern to implement ```getState()``` and ```loadState()``` methods:

```java
class MyService implements PersistentStateComponent<MyService> {
  public String stateValue;

  public MyService getState() {
    return this;
  }

  public void loadState(MyService state) {
    XmlSerializerUtil.copyBean(state, this);
  }
}
```

### Implementing the State Class

The implementation of ```PersistentStateComponent``` works by serializing public fields, 
[annotated](https://github.com/JetBrains/intellij-community/tree/master/platform/util/src/com/intellij/util/xmlb/annotations) 
private fields and bean properties into an XML format. The following types of values can be persisted:
- numbers (both primitive types, such as int, and boxed types, such as Integer);
- booleans;
- strings;
- collections;
- maps;
- enums.

In order to exclude a public field or bean property from serialization, you can annotate the field or getter with the ```@com.intellij.util.xmlb.annotations.Transient``` annotation.

Note that the state class must have a default constructor. 
It should return the default state of the component (one used if there is nothing persisted in the XML files yet).

State class should have a ```equals``` method, but if it is not implemented, state objects will be compared by fields. 
If you write in Kotlin, use [Data](http://kotlinlang.org/docs/reference/data-classes.html).

### Defining the Storage Location

In order to specify where exactly the persisted values wiil be stored, you need to add a ```@State``` annotation to the ```PersistentStateComponent``` class. 
It has the following fields:

* ```name``` (required) - specifies the name of the state (name of the root tag in XML)
* One or more of ```@com.intellij.openapi.components.Storage``` annotations (required) - specify the storage locations for .ipr and  directory-based projects
* ```reloadable``` (optional) - if set to false, complete project reload is required when the XML file is changed externally and the state has changed.

The simplest ways of specifying the ```@Storage``` annotation are as follows:

* ```@Storage(id="other", file = StoragePathMacros.APP_CONFIG + "/other.xml")``` for application-level values
* ```@Storage(id="other", file = StoragePathMacros.PROJECT_FILE)``` for values stored in the project file (for .ipr based projects)
* ```@Storage(id = "dir", file = StoragePathMacros.PROJECT_CONFIG_DIR + "/other.xml", scheme = StorageScheme.DIRECTORY_BASED)})``` for values stored in the project directory (for directory-based projects)
* ```@Storage(id="other", file = StoragePathMacros.WORKSPACE_FILE)``` for values stored in the workspace file

The ```roamingType``` parameter of the ```@Storage``` annotation specifies the roaming type when the Settings Repository plugin is used.

The ```id``` parameter of the ```@Storage``` annotation can be used to exclude specific fields from serialization in specific formats. 
If you do not need to exclude anything, you can set the ```id``` to an arbitrary string value.

By specifying a different value for the ```file``` parameter, you can cause the state to be persisted in a different file.

If you need to specify where the values are stored when the directory-based project format is used, you need to add the second ```@Storage``` annotation with the scheme parameter set to StorageScheme.DIRECTORY_BASED, for example:


```java
@State(
    name = "AntConfiguration",
    storages = {
      @Storage(id = "default", file = StoragePathMacros.PROJECT_FILE),
      @Storage(id = "dir", file = StoragePathMacros.PROJECT_CONFIG_DIR + "/ant.xml", scheme = StorageScheme.DIRECTORY_BASED)
    }
)
```

## Customizing the XML Format of Persisted Values

Please consider to use annotation parameters only to achieve backward compatibility. 
Otherwise feel free to file issues about serialization cosmetics.

If you want to use the default bean serialization but need to customize the storage format in XML (for example, for compatibility with previous versions of your plugin or externally defined XML formats), you can use the ```@Tag```, ```@Attribute```, ```@Property```, ```@MapAnnotation```, ```@AbstractCollection``` annotations. 
You can look at the source code (```com.intellij.util.xmlb``` package) to get more information about the meaning of these annotations.

If the state that you need to serialize doesn't map cleanly to a JavaBean, you can use ```org.jdom.Element``` as the state class. 
In that case, you can use the ```getState()``` method to build an XML element with an arbitrary structure, which will then be saved directly in the state XML file. 
In the ```loadState()``` method, you can deserialize the JDOM element tree using any custom logic. 
But this way is not recommended and should be avoided.

## Persistent Component Lifecycle

The ```loadState()``` method is called after the component has been created (only if there is some non-default state persisted for the component), and after the XML file with the persisted state is changed externally (for example, if the project file was updated from the version control system). In the latter case, the component is responsible for updating the UI and other related components according to the changed state.

The ```getState()``` method is called every time the settings are saved (for example, on frame deactivation or when closing the IDE). If the state returned from ```getState()``` is equal to the default state (obtained by creating the state class with a default constructor), nothing is persisted in the XML. Otherwise, the returned state is serialized in XML and stored.

## Legacy API (JDOMExternalizable)

Older IDEA components use the ```JDOMExternalizable``` interface for persisting state. 
It uses the ```readExternal()``` method for reading the state from a JDOM element, and ```writeExternal()``` to write the state to it. 
```JDOMExternalizable``` implementations can store the state in attributes and sub-elements manually, and/or use the ```DefaultJDOMExternalizer``` class to store the values of all public fields automatically.

When the component's class implements the ```JDOMExternalizable``` interface, the components save their state in the following files:

* Project-level components save their state to the project (.ipr) file. However, if the workspace option in the plugin.xml file is set to _true_, the component saves its configuration to the workspace (.iws) file instead.
* Module-level components save their state to the module (.iml) file.