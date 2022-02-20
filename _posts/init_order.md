Title: Roku: Component initialization order
Date: 2018.12.18
Summary: Understanding the component initialization order and some potential gotchas
Category: Patterns

Roku's documentation contains [a good article][RokuInitializationOrder] about component initialization order. Specifically, it states:

> Instances of components defined in an XML file follow a well-defined initialization order when they are created.
>
> - The `<children>` element nodes defined in XML markup are created, and their fields are set to their initial values, either to a default value, or to the value specified in the XML markup.
> - The `<interface>` element fields of the XML component are created, and their initial values are set, either to a default value, or to the value specified by the value attribute.
> - The `<script>` element `init()` function is called, and all initializations contained in the function are performed.

While that is very helpful, it leaves out some subtle bits and sometimes it can be hard to visualize, so let's do some experiments to better illustrate the order.

Consider the following component hierarchies:

    Parent > ParentBase > Group
    Child > ChildBase > Group

Let's let the `ParentBase` component define a `Child` in its `<children>` markup:

    ::: class="language-markup" :::
    <?xml version="1.0" encoding="utf-8" ?>
    <component name="ParentBase" extends="Group">
        <script type="text/brightscript" uri="pkg:/components/ParentBase.brs"/>
        <children>
            <Child id="childInParentBase"/>
        </children>
    </component>

And, let's imagine that the `Parent` component defines a `Child` component as part of its `<children>`:

    ::: class="language-markup" :::
    <?xml version="1.0" encoding="utf-8" ?>
    <component name="Parent" extends="ParentBase">
        <script type="text/brightscript" uri="pkg:/components/Parent.brs"/>
        <children>
            <Child id="childFromParent"/>
        </children>
    </component>

Now consider another component that uses the `Parent` component and defines some additional children:

    ::: class="language-markup" :::
    <?xml version="1.0" encoding="utf-8" ?>
    <component name="OtherComponent" extends="Group">
        <script type="text/brightscript" uri="pkg:/components/OtherComponent.brs"/>
        <children>
            <Parent id="parent">
                <Child id="firstchild"/>
                <Child id="secondchild"/>
            </Parent>
        </children>
    </component>

If you run this code, you will see that the order of the `init` calls is actually:

    ChildBase for childInParentBase
    Child for childInParentBase
    ParentBase for parent
    ChildBase for childFromParent
    ChildBase for childFromParent
    Parent for parent
    ChildBase for firstChild
    Child for firstChild
    ChildBase for secondChild
    Child for secondChild

So if we wanted to write a more complete order of initialization, it might look like this:

1. If the component extends another component, the base component (including all children) will be initialized first
2. Initialize nodes defined in `<children>`
3. Call component's `init()` method
4. Field values are available for reading
5. Children added via markup are initialized

Understanding the order is helpful, but understanding the implications is also very important. Consider these perhaps subtle implications:

### Non-default field values are not available in `init()`

Consider this `init()` code:

    sub init()
        ?"My id is: " + m.top.id
    end sub

Assuming the `id` property was set in markup, you might be surprised to see the output of this is:

    My id is:

At the time that `init()` is called, any non-default field values are not yet set, so `m.top.id` has no value.

Similarly, any values you set on `m.top` in `init()` could be overwritten when initialization is complete:

    sub init()
        m.top.id = "set_in_init"
        ?"My id is: " + m.top.id
    end sub

    function onKeyEvent(key, press)
        ?"My id is now: " + m.top.id
        return true
    end function

Prints:

    My id is: set_in_init
    My id is now: parent

### Cannot access `getParent()` from `init()`

The Roku docs say:

> For nodes that are defined in the `<children>` XML markup of the component file, the parent node is set after the node is created, and `init()` is called.

Essentially that means that the children are not yet 'parented' when their `init()` function is ran. That means that if you call `getParent()` in `init()`, it will return `invalid`.

### Cannot `setFocus()` from `init()`

Following on the above behavior, that means that the children created in a component are not yet valid targets for focus while `init()` is running. The children are not parented (added to the Scene Graph tree) until after `init()` is complete and calls to `setFocus()` require that the node being focused is in the Scene Graph tree. **However**, experimentation has shown that you can often get away with calling `setFocus()` on a child in the `init()` and I personally have done it often in real apps. I have also seen it not work and struggled to figure out why, so it is probably better to just avoid the ambiguity and not do this.

(It is usually not a good idea to set the focus during object creation anyway, and a better approach is to set up an observer on `focusedChild` so that your component is aware when something is trying to set focus on it and handle focus at that time.)


[RokuInitializationOrder]: https://sdkdocs.roku.com/display/sdkdoc/Component+Initialization+Order
