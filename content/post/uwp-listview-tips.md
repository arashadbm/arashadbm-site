+++
date = "2017-05-28T16:23:55+02:00"
description = ""
tags = []
title = "Uwp ListView Tips And Common Mistakes"
topics = []

+++

## Great apps have something in common - responsiveness:
Performance in apps is similar to what Magicians do, most of the time it's about what user perceives. Windows 10 applications are no exception, it's important for your app to be responsive. Apps that don't lag or take lots of time to load page, deliver high better user experience and hence users will love to use them. 

I'll share with you in this article some of the tips I learnt while developing multiple Windows apps in the past few years, these tips will help improve responsiveness of pages with ListView/GridView. You may already know some of them and few are new to you, so feel free to skim through what you already know.

### Use items panels with virtualization
When you use ListView or GridView without overriding ItemsPanelTemplate, you already have UI Virtualization by default, and only the items visible in the current viewport (plus few more**) will be loaded, when user scrolls through the list new items will be realized. This feature is important if the size of collection is big.

Default items panel for ListView is [ItemsStackPanel] (https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.itemsstackpanel) and [ItemsWrapGrid] (https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.itemswrapgrid) for GridView. So if you are overriding ItemsPanelTemplate and your collection will be big Make sure you use them and not the normal panels like StackPanel, Grid and WrapGrid....

Read more on how Virtualization works in this [great article](https://blogs.msdn.microsoft.com/alainza/2014/09/03/listview-basics-and-virtualization-concepts/) at Msdn written by Alainza

Example:
```
<ListView>
    <ListView.ItemsPanel> 
        <ItemsPanelTemplate>
            <ItemsStackPanel Orientation="Horizontal" CacheLength="4" />  
        </ItemsPanelTemplate> 
    </ListView.ItemsPanel> 
</ListView> 
```

> **The CacheLength property specifies the size of the buffers for the off-screen items. You specify CacheLength in multiples of the current viewport size. For example, if the CacheLength is 4.0, 2 viewports worth of items are buffered on each side of the viewport.
You can set a smaller cache length to optimize startup time, or set a larger cache size to optimize scrolling performance. Item containers that are off-screen are created at a lower priority than those in the viewport.

Note (Not recommended to use):  [VirtualizingStackPanel](https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.virtualizingstackpanel) also supports virtualization but it's old (Windows 8) and [ItemsStackPanel] (https://docs.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.itemsstackpanel) beats it in terms of perfromance as it supports pixel-based UI virtualization and grouped layouts.


### Don't put ListView inside ScrollViewer

Before we go through the remaining tips, it’s important to make sure that we don’t kill virtualization.  Is it possible to kill it? Hell yes by putting ListView inside any control that that passes `double.Infinity` while measuring it's content's size like ScrollViewer.

Virtualization needs to have exact space to be able to operate, this space will be the viewport. Virtualization panels in this case loads items that are in viewport (plus few more to have smooth scrolling experience)

Trust me it seems easy but multiple developers fall in this trap a lot.
```
<!--DON'T USE THIS EXAMPLE, it will have Perfromance issues-->
<ScrollViewer>
   <ListView/>
</ScrollViewer>
```
### Don't put ListView inside StackPanel or RowDefinition=Auto
Similar trap to the previous ScrollViewer is to put ListView inside GridView in a row with `<RowDefinition Height="Auto" />` or StackPanel, which in this case also passes Infinity while measuring ListView's size and arranging it. This time you lose both Virtualization and Scrolling ability.

```
<!--DON'T USE THIS EXAMPLE, it will have Scrolling and Perfromance issues-->
<StackPanel>
   <ListView/>
</StackPanel>

<!--DON'T USE THIS EXAMPLE, it will have Scrolling and Perfromance issues-->
<Grid>
   <Grid.RowDefinitions>
      <RowDefinition Height="Auto" />
      <RowDefinition Height="*" />
   </Grid.RowDefinitions>
   <ListView Grid.Row="0" />
</Grid>   
```

### Use XBinding

Use xBinding in item template whenever possible as it's resolved at compile time, so no reflection is needed which gives performance advantage and it uses less memory than normal binding.
If you are binding to properties that won't change like commands, Use x:Binding with one time binding instead of OneWay.

Example: 
``` Xml
<DataTemplate x:DataType="mode:Person">
 <StackPanel>
   <TextBlock Text="{x:Bind FirstName}"/>
   <TextBlock Text="{x:Bind Age}"/>
 </StackPanel>
</DataTemplate>
```
Notes:

* In x:Bind, Path is rooted at the Page by default, not the DataContext.
* Default binding mode is OneTime in XBinding.
* In x:Binding, When there is a null in Path evaluated value, it raises exceptions not like  normal binding which was showing warnings in debug console.

You can read more about binding in depth in this article from Microsoft, 
[here] (https://docs.microsoft.com/en-us/windows/uwp/data-binding/data-binding-in-depth) .

### Use XPhase
Now that you use x:Bind, you can use x:Phase to incrementaly load items while user is scrolling  inside ListView/GridView, it gives the user the feeling of responsiveness if he is scrolling fast as the loading won't be completed of items that went out of viewport.

A quote from Microsoft [documentation] (https://docs.microsoft.com/en-us/windows/uwp/xaml-platform/x-phase-attribute):

>When an element has a phase other than 0, the element will be hidden from view (via Opacity, not Visibility) until that phase is processed and bindings are updated. When a ListViewBase-derived control is scrolled, it will recycle the item templates from items that are no longer on screen to render the newly visible items. UI elements within the template will retain their old values until they are data-bound again.

Example: 

``` Xml
<DataTemplate x:DataType="model:Person">
    <Grid Height="80">
        <Grid.ColumnDefinitions>
           <ColumnDefinition Width="80" />
           <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>
        <Image Source="{x:Bind ImageData}"  Grid.Column="0" " x:Phase="2"/>
        <TextBlock Text="{x:Bind DisplayName}" Grid.Column="1" FontSize="12"/>
    </Grid>
</DataTemplate>
```
### Reduce element count
Sometimes we can try to find all possible solutions to boost performance of list view but the main problem is how we structured ItemTemplate. Imagine you have item template with total item count 30, now the viewport shows around 10 items. So List view will have to render CacheLength * 30 * 10, with cache length of 2 for example, total = 600 element. Now if you revisited the design structure and removed those nested Grids or stacks that can be replace by RelativePanel or vice versa and reduce item template's element count to only 15. Now ListView needs to manage only 300 elements instead of 600.

Okay so how can I find element count without all of these calculations?
By using [Visual Live Tree inspector](https://msdn.microsoft.com/en-us/library/mt270227.aspx) which is part of Visual studio. 

![Visual Live Tree](/img/visual_live_tree.png)