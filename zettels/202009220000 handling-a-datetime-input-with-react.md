
---
title: "Dealing with a datetime input in React?"
slug: "handling-a-datetime-input-with-react"
date: "2020-09-22"
summary: ""
---

#software-development #web-development #react

In order to collect a timestamp using React I have long been searching for components which could help me to do so. There are various libraries available, but ironically none of them help me more than the browsers built in `datetime-local` input type.

So I went and implemented this input type, however, there were some complications regarding data updates, especially when changing the date. I have written out a fairly basic but nice working component with which I can collect a datetime.


An important practice within front-end development for me is to directly put a data value within the object I use to communicate with the API. For me this means that I should return a date format which is understood by the server, which isn't the case with the default format.

> I'm using MomentJS for some normalization tasks. At the time of writing MomentJS has just entered maintenance mode and isn't being expanded any further. Therefore I wouldn't recommend including it in your project if you randomly stumble upon this code snippet.

As more often than not, this work is directly related to the stuff I'm doing with [Skyhop](https://skyhop.org). The datetime component, when implemented, looks like this;

![A form meant to enter (flight) departure information with two elements (departure airfield, and departure time), grouped visually together with an inner shadow.](/uploads/Screenshot_2020_09_22_103643_aaae52bc42.jpg)


Some noteworthy details about this snippet;

1. It accepts a value prop, but will keep an internal representation for the input's value. Updating the prop will update the component. Updating the component will trigger the `onChange` method.
2. I have implemented a button to quickly set the value to the current moment.
3. I assume the value is an ISO compatible date time format.

```jsx
import React, { useState } from 'react'
import moment from 'moment'

export function DateTimeInput({ value, onChange }) {
  const [_currentValue, _setCurrentValue] = useState(null)
  const [_value, _setValue] = useState("")

  /*
   * We're handling the datetime-local translation internally, and only output valid datetime objects
   */
  if (_currentValue !== value) {
    _setCurrentValue(value)
    if (value != null && value != "") {
      _setValue(moment(value).local().format(moment.HTML5_FMT.DATETIME_LOCAL))
    }
  }

  return (
    <>
      <div>
        <label htmlFor="dt" className="block text-sm font-normal leading-5 text-gray-500">Time</label>
        <div className="mt-1 flex rounded-md shadow-sm">
          <div className="relative flex-grow">
            <input type="datetime-local" value={_value}
              onChange={(e) => {
                _setValue(e.target.value)

                const currentValue = moment(e.target.value)
                if (currentValue.isValid() && onChange != null) {
                  onChange(currentValue.utc().toISOString())
                }
              }}
              id="dt"
              className="form-input block w-full rounded-none rounded-l-md pl-3 transition ease-in-out duration-150 sm:text-sm sm:leading-5" placeholder="Departure Time" />
          </div>
          <button
            onClick={(e) => {
              e.preventDefault()
              const val = moment().local().format(moment.HTML5_FMT.DATETIME_LOCAL)
              _setValue(val)

              if (onChange != null) {
                onChange(moment(val).utc().toISOString())
              }
            }}
            className="-ml-px relative inline-flex items-center px-4 py-2 border border-gray-300 text-sm leading-5 font-medium rounded-r-md text-gray-700 bg-gray-50 hover:text-gray-500 hover:bg-white focus:outline-none focus:shadow-outline-blue focus:border-blue-300 active:bg-gray-100 active:text-gray-700 transition ease-in-out duration-150">
            Now
          </button>
        </div>
      </div>
    </>
  )
}
```


