---
title: "Creating Platform-Specifics"
description: "Vendors can create their own platform-specifics with Effects. An Effect provides the specific functionality, which is then exposed through a platform-specific. The result is an Effect that can be more easily consumed through XAML, and through a fluent code API. This article demonstrates how to expose an Effect through a platform-specific."
ms.prod: xamarin
ms.assetid: 0D0E6274-6EF2-4D40-BB77-3D8E53BCD24B
ms.technology: xamarin-forms
author: davidbritch
ms.author: dabritch
ms.date: 11/23/2016
---

# Creating Platform-Specifics

_Vendors can create their own platform-specifics with Effects. An Effect provides the specific functionality, which is then exposed through a platform-specific. The result is an Effect that can be more easily consumed through XAML, and through a fluent code API. This article demonstrates how to expose an Effect through a platform-specific._

## Overview

The process for creating a platform-specific is as follows:

1. Implement the specific functionality as an Effect. For more information, see [Creating an Effect](~/xamarin-forms/app-fundamentals/effects/creating.md).
1. Create a platform-specific class that will expose the Effect. For more information, see [Creating a Platform-Specific Class](#creating).
1. In the platform-specific class, implement an attached property to allow the platform-specific to be consumed through XAML. For more information, see [Adding an Attached Property](#attached_property).
1. In the platform-specific class, implement extension methods to allow the platform-specific to be consumed through a fluent code API. For more information, see [Adding Extension Methods](#extension_methods).
1. Modify the Effect implementation so that the Effect is only applied if the platform-specific has been invoked on the same platform as the Effect. For more information, see [Creating the Effect](#creating_the_effect).

The result of exposing an Effect as a platform-specific is that the Effect can be more easily consumed through XAML and through a fluent code API.

> [!NOTE]
> It's envisaged that vendors will use this technique to create their own platform-specifics, for ease of consumption by users. While users may choose to create their own platform-specifics, it should be noted that it requires more code than creating and consuming an Effect.

The sample application demonstrates a `Shadow` platform-specific that adds a shadow to the text displayed by a [`Label`](https://developer.xamarin.com/api/type/Xamarin.Forms.Label/) control:

![](creating-images/screenshots.png "Shadow Platform-Specific")

The sample application implements the `Shadow` platform-specific on each platform, for ease of understanding. However, aside from each platform-specific Effect implementation, the implementation of the Shadow class is largely identical for each platform. Therefore, this guide focusses on the implementation of the Shadow class and associated Effect on a single platform.

For more information about Effects, see [Customizing Controls with Effects](~/xamarin-forms/app-fundamentals/effects/index.md).

<a name="creating" />

## Creating a Platform-Specific Class

A platform-specific is created as a `public static` class:

```csharp
namespace MyCompany.Forms.PlatformConfiguration.iOS
{
  public static Shadow
  {
    ...
  }
}
```

The following sections discuss the implementation of the `Shadow` platform-specific and associated Effect.

<a name="attached_property" />

### Adding an Attached Property

An attached property must be added to the `Shadow` platform-specific to allow consumption through XAML:

```csharp
namespace MyCompany.Forms.PlatformConfiguration.iOS
{
	using System.Linq;
	using Xamarin.Forms;
	using Xamarin.Forms.PlatformConfiguration;
	using FormsElement = Xamarin.Forms.Label;

	public static class Shadow
	{
		const string EffectName = "MyCompany.LabelShadowEffect";

		public static readonly BindableProperty IsShadowedProperty =
			BindableProperty.CreateAttached("IsShadowed",
			                                typeof(bool),
			                                typeof(Shadow),
			                                false,
			                                propertyChanged: OnIsShadowedPropertyChanged);

		public static bool GetIsShadowed(BindableObject element)
		{
			return (bool)element.GetValue(IsShadowedProperty);
		}

		public static void SetIsShadowed(BindableObject element, bool value)
		{
			element.SetValue(IsShadowedProperty, value);
		}

        ...

		static void OnIsShadowedPropertyChanged(BindableObject element, object oldValue, object newValue)
		{
			if ((bool)newValue)
			{
				AttachEffect(element as FormsElement);
			}
			else
			{
				DetachEffect(element as FormsElement);
			}
		}

		static void AttachEffect(FormsElement element)
		{
			IElementController controller = element;
			if (controller == null || controller.EffectIsAttached(EffectName))
			{
				return;
			}
			element.Effects.Add(Effect.Resolve(EffectName));
		}

		static void DetachEffect(FormsElement element)
		{
			IElementController controller = element;
			if (controller == null || !controller.EffectIsAttached(EffectName))
			{
				return;
			}

			var toRemove = element.Effects.FirstOrDefault(e => e.ResolveId == Effect.Resolve(EffectName).ResolveId);
			if (toRemove != null)
			{
				element.Effects.Remove(toRemove);
			}
		}
	}
}
```

The `IsShadowed` attached property is used to add the `MyCompany.LabelShadowEffect` Effect to, and remove it from, the control that the `Shadow` class is attached to. This attached property registers the `OnIsShadowedPropertyChanged` method that will be executed when the value of the property changes. In turn, this method calls the `AttachEffect` or `DetachEffect` method to add or remove the effect based on the value of the `IsShadowed` attached property. The Effect is added to or removed from the control by modifying the control's [`Effects`](https://developer.xamarin.com/api/property/Xamarin.Forms.Element.Effects/) collection.

> [!NOTE]
> Note that the Effect is resolved by specifying a value that's a concatenation of the resolution group name and unique identifier that's specified on the Effect implementation. For more information, see [Creating an Effect](~/xamarin-forms/app-fundamentals/effects/creating.md).

For more information about attached properties, see [Attached Properties](~/xamarin-forms/xaml/attached-properties.md).

<a name="extension_methods" />

### Adding Extension Methods

Extension methods must be added to the `Shadow` platform-specific to allow consumption through a fluent code API:

```csharp
namespace MyCompany.Forms.PlatformConfiguration.iOS
{
	using System.Linq;
	using Xamarin.Forms;
	using Xamarin.Forms.PlatformConfiguration;
	using FormsElement = Xamarin.Forms.Label;

	public static class Shadow
	{
        ...
		public static bool IsShadowed(this IPlatformElementConfiguration<iOS, FormsElement> config)
		{
			return GetIsShadowed(config.Element);
		}

		public static IPlatformElementConfiguration<iOS, FormsElement> SetIsShadowed(this IPlatformElementConfiguration<iOS, FormsElement> config, bool value)
		{
			SetIsShadowed(config.Element, value);
			return config;
		}
        ...
	}
}
```

The `IsShadowed` and `SetIsShadowed` extension methods invoke the get and set accessors for the `IsShadowed` attached property, respectively. Each extension method operates on the `IPlatformElementConfiguration<iOS, FormsElement>` type, which specifies that the platform-specific can be invoked on [`Label`](https://developer.xamarin.com/api/type/Xamarin.Forms.Label/) instances from iOS.

<a name="creating_the_effect" />

### Creating the Effect

The `Shadow` platform-specific adds the `MyCompany.LabelShadowEffect` to a [`Label`](https://developer.xamarin.com/api/type/Xamarin.Forms.Label/), and removes it. The following code example shows the `LabelShadowEffect` implementation for the iOS project:

```csharp
[assembly: ResolutionGroupName("MyCompany")]
[assembly: ExportEffect(typeof(LabelShadowEffect), "LabelShadowEffect")]
namespace ShadowPlatformSpecific.iOS
{
	public class LabelShadowEffect : PlatformEffect
	{
		protected override void OnAttached()
		{
			UpdateShadow();
		}

		protected override void OnDetached()
		{
		}

		protected override void OnElementPropertyChanged(PropertyChangedEventArgs args)
		{
			base.OnElementPropertyChanged(args);

			if (args.PropertyName == Shadow.IsShadowedProperty.PropertyName)
			{
				UpdateShadow();
			}
		}

		void UpdateShadow()
		{
			try
			{
				if (((Label)Element).OnThisPlatform().IsShadowed())
				{
					Control.Layer.CornerRadius = 5;
					Control.Layer.ShadowColor = UIColor.Black.CGColor;
					Control.Layer.ShadowOffset = new CGSize(5, 5);
					Control.Layer.ShadowOpacity = 1.0f;
				}
				else if (!((Label)Element).OnThisPlatform().IsShadowed())
				{
					Control.Layer.ShadowOpacity = 0;
				}
			}
			catch (Exception ex)
			{
				Console.WriteLine("Cannot set property on attached control. Error: ", ex.Message);
			}
		}
	}
}
```

The `UpdateShadow` method sets `Control.Layer` properties to create the shadow, provided that the `IsShadowed` attached property is set to `true`, and provided that the `Shadow` platform-specific has been invoked on the same platform that the Effect is implemented for. This check is performed with the `OnThisPlatform` method.

If the `Shadow.IsShadowed` attached property value changes at runtime, the Effect needs to respond by removing the shadow. Therefore, an overridden version of the `OnElementPropertyChanged` method is used to respond to the bindable property change by calling the `UpdateShadow` method.

For more information about creating an effect, see [Creating an Effect](~/xamarin-forms/app-fundamentals/effects/creating.md) and [Passing Effect Parameters as Attached Properties](~/xamarin-forms/app-fundamentals/effects/passing-parameters/attached-properties.md).

## Consuming a Platform-Specific

The `Shadow` platform-specific is consumed in XAML by setting the `Shadow.IsShadowed` attached property to a `boolean` value:

```xaml
<ContentPage xmlns:ios="clr-namespace:MyCompany.Forms.PlatformConfiguration.iOS" ...>
  ...
  <Label Text="Label Shadow Effect" ios:Shadow.IsShadowed="true" ... />
  ...
</ContentPage>
```

Alternatively, it can be consumed from C# using the fluent API:

```csharp
using Xamarin.Forms.PlatformConfiguration;
using MyCompany.Forms.PlatformConfiguration.iOS;

...

shadowLabel.On<iOS>().SetIsShadowed(true);
```

For more information about consuming platform-specifics, see [Consuming Platform-Specifics](~/xamarin-forms/platform/platform-specifics/consuming/index.md).

## Summary

This article demonstrated how to expose an Effect through a platform-specific. The result is an Effect that can be more easily consumed through XAML, and through a fluent code API.


## Related Links

- [ShadowPlatformSpecific (sample)](https://developer.xamarin.com/samples/xamarin-forms/userinterface/shadowplatformspecific/)
- [Customizing Controls with Effects](~/xamarin-forms/app-fundamentals/effects/index.md)
- [Attached Properties](~/xamarin-forms/xaml/attached-properties.md)
