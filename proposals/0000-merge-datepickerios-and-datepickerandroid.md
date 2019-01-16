# RFC0000: Merge DatePickeriOS and DatePickerAndroid

## Summary

React Native has a multiple Components that deal with date and time on both iOS and Android. The goal of this 
document is to propose a merge between `DatePickerIOS` and both components on Android (`DatePickerAndroid` and 
`TimePickerAndroid`). In addition this single module should become a react community module. Both platforms 
use different approaches with respect to both configurability and implementation. 

First, this proposal will outline differences and similarities between Android and iOS. Next, it should 
discuss a unified API to make the module easier to maintain and allow for more (new) features to be added. 
Anything in this proposal is open to discussion. Alternative approaches are welcome!

The UI will remain similar to what native iOS or Android currently provide. This proposal **does not** aim to
redefine the UIs.

## Basic Example

```js
class Calendar extends React.Component {
  state = {
    date: new Date()
  };

  render() {
    return <DateTimePicker
      value={ this.state.date }                             
      onDateChange={ date => this.setState({ date }) }      // date is `Date` object
      mode='date'                                           // new default
    />
  }
}
```

## Motivations

Deprecate the existing modules `DatePickerIOS`, `DatePickerAndroid` and `TimePickerAndroid` as outlined 
by the ['Lean Core' proposal][lean].

## Prior Art

Before digging into the new implementation proposal let's examine the previous implementations across both platforms.
The biggest difference between both platforms is a Component interface on iOS vs. an API interface on Android. 
In addition, the date and time picker for Android are split. This is congruous to the native implementations 
which are also split for Android, e.g. `TimePickerDialog` and `DatePickerDialog`.

Next, iOS and Android have different implementations for event handling and state management. 
[iOS][ios] is based on a controlled component that requires `props.onDateChange` to be passed. Otherwise the 
interacting user wouldn't be able to select a date. Android however is returning a `Promise`. This promise needs to be 
acted on after it resolves. This is different from the common React paradigm that uses event handlers. 
In addition, Android requires checking the `action` property whenever the `Promise` resolves.

> If the user dismissed the dialog, the Promise will still be resolved with action being 
> `DatePickerAndroid.dismissedAction` and all the other keys being undefined.

Both platforms require `props.date` to be set to select a date other than the current. i.e. there is no 
difference in using the default date or setting a specific date from state. State is also not retained on
Android when the calendar is dismissed.

The default `mode` is `datetime` for iOS. This default is [specified in JS][datetime] through `defaultProps`. 
Obviously this `mode` is [not available for android][open-android] as it depends on separate Components. 
The invocation of `DatePickerAndroid` and `TimePickerAndroid` is very similar. The default `mode` for both components 
is `default`, which matches the default implementation of the running Android version. 

Finally, iOS allows `date` to be a `timestamp` or `Date` object. `DatePickerAndroid` matches this 
implementation. The `TimePickerAndroid` however requires a `number` to be passed to the `hour` and `minute` props. 

---

## Detailed design

Deprecate the existing modules `DatePickerIOS`, `DatePickerAndroid` and `TimePickerAndroid` as outlined 
by the ['Lean Core' proposal][lean]. The modules will be replaced with a new single Component 
called `DateTimePicker`. Which will support a set of properties that cover configurability for both Android
and iOS. A simple example implementation would look like: 

```js
class Calendar extends React.Component {
  state = {
    date: new Date()
  };

  render() {
    return <DateTimePicker
      value={ this.state.date }                             
      onDateChange={ date => this.setState({ date }) }      // date is `Date` object
      mode='date'                                           // new default
    />
  }
}
```

This requires a new component called `DateTimePickerAndroid` to be created. It will effectively merge 
the API's of `DatePickerAndroid` and `TimePickerAndroid`. It wraps the function calls required for Android to create 
a React component. It could be a functional component, e.g.

```js
function DateTimePickerAndroid({ mode, onDateChange, value, ...props }) {
  const Selector = mode === 'time' ? TimePickerAndroid : DatePickerAndroid;

  Selector.open({ ...value, ...props }) // act on thenable with `onDateChange`
}
```

### Naming

The proposed name is `react-native-datetimepicker`, with a Github repo in the react native community 
`git@github.com:react-native-community/react-native-datetimepicker.git`. However, suggestions are welcome!

### Options 

Property names should remain similar where possible. However there are some difference between android and iOS
that need to be resolved. This section will talk about required changes.

#### value

Both `DatePickerIOS` and `DatePickerAndroid` use `date` whereas `TimePickerAndroid` requires an `hour` and 
`minute` property. The proposal is to change this property to `value {Date, Number}`. Where `{Number}` is
a unix epoch timestamp. Both [Android][month-android] and [JS][month] use a month integer from `[n](0,11)`, 
making conversions from and to `Date` relatively easy. Also converting a `Date` to `timestamp` is simple (iOS
requires timestamps). `Date` also supports setting the `hour` and `minute` making it an ideal candidate to
support all native implementations.

Finally, using a more generic description for `date` and `time` allows it to be [used for setting
`countDownDuration`][countdown]. Enabling the [feature implementation of `mode: countdown`](#mode) or
doing a [calculation of `timeZoneOffsetInMinutes` in JS](#timezoneoffsetinminutes-ios).

#### onDateChange

The `eventHandler` will function on iOS as before and ensure the controlled component works as expected. 
Android will act on the returned `Promise`.

```js
<DatePickerAndroid, TimePickerAndroid>.then(function resolve({ action, day, month, year }) {
  if (action === 'dateSetAction') onDateChange(new Date(year, month, day));
  if (action === 'dismissedAction') onDateChange();
}, function reject(error) {
  // ignore or throw `activity == null` error
});
```

Note: the [`activity == null` error rejection][reject] requires a solution. This `Error` could either be thrown to
let it surface as red screen or ignored entirely. 

#### maximumDate & minimumDate

Use the `iOS` property called `maximumDate` to constraint date ranges. This implies `maxDate` from
Android is renamed to `maximumDate`. The same approach will be used to rename the `minDate` to `minimumDate`.
These options should be ignored for `TimePickerAndroid`.

#### mode 

The current `mode` should be split up in two different properties, `mode` and `display`. The default for
`mode` should be changed to `date`. Both iOS specific options would render the default on Android. 
Support for platform specific implementations can stay the same. For example, `mode: datetime` would be 
ignored for Android. Available values for `mode`:

- `date` (new default for both android and iOS)
- `time`
- `datetime (iOS)` 
- `countdown (iOS)`

_The availability of `mode: countdown` for iOS is currently not part of the documentation since it is not
completely implemented yet._

#### display (android)

The `display` property will control the visual display of the picker for Android and will be ignored for iOS.
There is no [configurability for the appearance of the iOS `UIDatePicker`][appearance]. Available values for
`display`:

- `default` 
- `spinner`
- `calendar` 
- `clock`

#### timeZoneOffsetInMinutes (iOS)

This feature is iOS only and can be ported over without change. However it could be added to Android as well if the
`value` property is based on a `Date` or `timestamp` object. Both platforms use GMT as the base for timezone
offsets. It would require some manual calculation in JS to pass the correct values to the Android `TimePicker` API. 
Timezone offsets culd still be calculated in minutes relative to GMT. For example, to set the `+1h` timezone 
(CEST) the value `60` (minutes) should be passed.

#### locale (iOS)

This property will remain iOS only and implemented as-is. Android has no support to change the `locale` on-the-fly.

#### is24Hour (android)

Setting the picker to use a 24hour format is and should only be available for Android. iOS uses `locale` to determine 
to correct format. Forcefully overriding the `locale` on iOS to show a 24 hour timer is not an option. Also it is 
already possible through `locale` if the implementing developer requires this behavior.

## Drawbacks

Given this was outlined as part of the ['Lean Core' proposal][lean] the same drawbacks of 'Lean Core' apply to this proposal. 'Lean Core' ftw though.

## Alternatives

The other clear alternative is to use the API-based DateTimePicker instead of a Component based one. However since Components are the most easily understood concept for React developers the choice to prefer the Component-based approach was clear.

## Adoption strategy

This would be a breaking change that would require updates from existing React Native applications. As such it is prudent to:

1. Complete work on a decoupled `react-native-datetimepicker` as defined above.
   - `UPGRADE.md` within that repository should outline in detail how to 
     migrate.
2. Deprecate `DatePickerIOS`, `DatePickerAndroid` and `TimePickerAndroid` for one or more releases of `react-native`. These deprecation notices should clearly indicate `react-native-datetimepicker` as the blessed alternative.
3. Remove `DatePickerIOS`, `DatePickerAndroid` and `TimePickerAndroid` after being deprecated for a sufficient time. 

## How we teach this

There are no new concepts introduced in this proposal. The bulk of the teaching would would live in `UPGRADE.md` within `react-native-datetimepicker`.

## Unresolved questions

There are no open questions at this time.

[datetime]: https://github.com/facebook/react-native/blob/master/Libraries/Components/DatePicker/DatePickerIOS.ios.js#L117-L119
[month]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date#Parameters
[reject]: https://github.com/facebook/react-native/blame/master/ReactAndroid/src/main/java/com/facebook/react/modules/datepicker/DatePickerDialogModule.java#L114-L119
[ios]: https://facebook.github.io/react-native/docs/datepickerios
[open-android]: https://facebook.github.io/react-native/docs/timepickerandroid#open
[countdown]: https://developer.apple.com/documentation/uikit/uidatepicker/1615986-countdownduration
[appearance]: https://developer.apple.com/documentation/uikit/uidatepicker#2281468
[month-android]: https://developer.android.com/reference/java/util/Calendar#MONTH
[lean]: https://github.com/react-native-community/discussions-and-proposals/issues/6
