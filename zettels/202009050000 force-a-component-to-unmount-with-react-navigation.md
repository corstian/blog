
---
title: "Force a component to unmount with React Navigation"
slug: "force-a-component-to-unmount-with-react-navigation"
date: "2020-09-05"
summary: ""
---

#software-development #react

While developing a realtime feature within a React Native app I discovered a specific quirk within the React Native Navigation library which leaves components mounted, even after the active route has changed. Based on the documentation I believe this has been an intentional design decision, though I couldn't find any specifics about this.

The issue manifested itself when working with a GraphQL subscription which kept receiving data even after one navigated away from the page (component). This resulted in an unnecessary CPU and memory drain and impacted performance of the application. As stated within the Apollo GraphQL documentation, the `useSusbcription` hook should automatically unsubscribe once the component is unloaded.


Additionally, React Native Navigation introduces an additional hook which represent two lifetime events within the framework, named `useFocusEffect`.

In retrospect the solution to my specific issue seems embarrassing easily, though it took me half a day to figure out. It involves in rendering some JSX based on a variable set using this custom `useFocusEffect` hook;

```js
import React, { useState, useCallback } from 'react'
import { useFocusEffect } from '@react-navigation/native'

export default function Live() {
  const [isVisible, setIsVisible] = useState(false)

  useFocusEffect(
    useCallback(() => {
      setIsVisible(true)

      return () => {
        setIsVisible(false)
      }
    }, [])
  )

  return (
    <>
      {isVisible && 
        <>
          {/* Your component goes here */}
        </>
      }
    </>
  )
}
```
