---
layout: post
title:  "Model based Testing with Fast-Check"
date:   2023-11-28 12:19:54 +0200
categories: Testing
---

<style>

  @media(min-width:1000px){
    .wrapper{
      max-width:80%;
    }

  }

  details{
    margin-bottom:36px;
  }


</style>


In this post, we will explore an example of model based testing (mbt) with fastcheck in combination with playwright, for a date range picker component. 

### Model based testing basics
Model based testing is a form of testing where actions are simulated with the goal of finding errors that happen after a specific sequence of actions. It is a good testing approach for any system under test (sut) where the state or conditions dictate multiple possible actions an actor (a user or some control system) can do, and the state or output of the sut is dependent on previous action. After each action a model (that you need to build) is used to check the state of the sut / result of an action against; hence the name model based testing. This model is a simplified representation of the sut. 

A self scan counter at the supermarket would be an example of something you can test with mbt. You can start scanning an article, add more articles, remove one, add your bonus card, start the payout, maybe get checked, etc. A text editor is also one: you can type, remove text, push the bold button, paste stuff in etc. In both these cases there are multiple actions available, the result will not be  one fixed answer or state (if there even is some final state) but is dependend on what happened previously and where you stop.  Any new state can open up or close off possible actions. 

These type of suts are hard to test because there are many sequences of actions that need to be tested and it is hard to write one testcase for each and every one. Model based testing will help out then by running many simulation, each of wich consisting of some randomized sequence of actions. So it will execute many paths through the sut and build more trust / find more possible bugs. 

For a more thorough definition and flavors of mbt, I recommend this [article](https://www.guru99.com/model-based-testing-tutorial.html). 


Lets first take a look on the example and the setup around it, before diving deeper into the mbt with fastcheck

### The example: a date range picker
For this example I would like to explore the fastcheck library on some front-end component with playwright, as a proof of concept for some harder components like a text editor. So we need something with multiple actions, that has some state, but not to complex since this is only a try-out. A date range picker component seems a good starting point. There is enough actions a user can take (browse between months, year, pick start date and or end date, close the picker, open it either from the start date or the end date etc). Availability of some of the actions are dependent what you did before. And there is many ways to end up with a particular selected date range.  

The [carbon component library](https://github.com/carbon-design-system/carbon-for-ibm-dotcom/tree/main/packages/carbon-web-components) has some nice looking components to use in your front-end ui and more importantly for this example, it has a webcomponent for this date range picker.  That saves having to deal with setting up a framework and comp- and/or transpiling issues. 


##  The general idea and setup
### General idea

To enable model based testing, we need to do the following:

- *write code to perform the actions in the sut (the date range picker)*. The user-actions in the date range picker are pretty straight forward: open the calendar, select a day, move forward or backward by month or year etc. Playwright will be used to implement these actions. We will also need to implement some functions to extract state from the component, like the selected date range, so we can check this against the model.

- *write a simplified model to check the outcome against* 
Then we must build a model that implements the same methods but in a simpler way than the date range picker itself. It must also keep track of what dates are selected, what month is displayed on the calendar, etc. This model will be a typescript class. 

- *Defining the commands for fastcheck.*
Once we know how to do an action in the sut and model, we need to feed fastcheck with a class that performs both actions, and asserts on the outcome. It is simply a mapping from some generalised command (like select a day) to the right actions in both the sut and the model. In this example, it will be pretty much a one-to-one mapping from action to a command. 

- *Setting up fastcheck for a run.*
Then, by following the fastcheck apis, we should be able to run a simple test in which fastcheck chooses commands and compares the results from the model and the sut.


### Tooling used
This is a example of mbt in fastcheck of a front-end component with playwright, so we are obviously going to use fastcheck and playwright. Playwright needs to be able to open the page with the date range picker. This a standalone html page with just the component, and the component source code is downloaded from some external cdn in a script tag. So there is not even a need to serve it locally: It can be opened as a local file (this is not a production like test example).  

So the tooling is pretty simple, I wrote a simple html file that displays the date range picker, there is node/npx to run the tests and to run playwright, playwright to do the automation for the date range picker and fast-check to do the mbt. (When I say playwright I mean playwright + playwright-test, because with playwright-test you get all kinds of extra's like video recording an ui-tracing that are handy for debugging ootb). 


## Zooming in
### Actions in the sut: Page Object model
One nice way to organise test code for e2e /ui tests is the [page object model](https://playwright.dev/docs/pom)(pom). The goal of the pom is to describe the actions a user can do with the page and implement them (in this case) using playwright. Also it should have methods that use playwright to extract state from the page for checking the outcome. It is a nice way to organise and re-use code and hide some of the playwright logic.  Now if you are making heavy use of components, a natural extension would be to also use component object models (com), since components are going to be used across multiple pages and can be re-used in tests also.   

It turns out the date-range-picker makes use of a date-picker-inputs components and opening one of the inputs actually opens another external library: [flatpickr](https://flatpickr.js.org/examples/). The flatpickr component is used to display the actual calendar. So I ended up with a com for a date range picker that uses two date-input components (a "from" input and a "to" input both defined by a com), and clicking either one of the inputs opens the flatpickr calendar, which will be the final com. In the details (🞂) below you can expand the code for each com


<details><summary markdown='span'>the flatpickr calendar</summary>
    

```typescript
export class FlatPickr{
  readonly page: Page  
  readonly me: Locator

  constructor(parent:Locator, page:Page){
    this.page = page
    this.me = parent.locator(".flatpickr-calendar")
  }

  async selectNextMonth(){
    await this.me.locator('.flatpickr-next-month').click()
  }
  async selectPrevMonth(){
    await this.me.locator('.flatpickr-prev-month').click()
  }
  async selectPrevYear(){
    await this.me.locator(".numInputWrapper .arrowDown").click()
  }
  async selectNextYear(){
    //unlike selectPrevYear implementation, clicking (with force) does not work for up button
    await this.me.locator(".numInputWrapper .arrowUp").dispatchEvent("mousedown")
  }

  async selectDay(day:string, month?:"prev"|"next"){
    
    let classSelector = ""
    if (month=== "prev"){
      classSelector = ".flatpickr-day.prevMonthDay"
    }else if (month ==="next"){
      classSelector=".flatpickr-day.nextMonthDay"
    }else{
      classSelector= ".flatpickr-day:not(.prevMonthDay):not(.nextMonthDay)"
    }
  
    const regex = new RegExp(`^${day}$`)    
    const dayToPick=  await this.me.locator(classSelector).filter({hasText:regex})
    const date = await dayToPick.getAttribute("aria-label")
    await dayToPick.click() 
  }
}

```

1. The flatpickr calendar is where most of the user interactions are defined. 
2. In the selectNextYear method click is not working on the flatpickr calendar for some reason. 
3. (The selectday is more complex since it needs to be because I only picked days displayed on the calendar for the current month, as we will see later on)

</details>

<details><summary markdown='span'>The date range range input</summary>

```typescript
export class DateInput{
  readonly parent: Locator  
  readonly kind: Kind
  readonly inputField: Locator
  isOpen: boolean = false

  constructor(parent:Locator, kind:Kind){
    this.kind = kind
    this.parent = parent
    this.inputField = parent.locator(`cds-date-picker-input[kind="${kind}"]`)
  }

  async myDate(){
    return await this.inputField.evaluate( (el:any)=>el.value)
  }

  async clickInputBox(){
    await this.inputField.getByRole("textbox").click()  
  }
}
  
```

1. constructor gets in the parent drp component and selects itself. 
2. There is one method to click the inputbox and one to read the date. 
3. plan was to also implement typing actions, but I did not get around to that. But in case I had, this class would be the right place for implementing it, I feel.  


</details>

<details><summary  markdown='span'>The date range picker</summary>

```typescript
import {  type Locator, type Page } from '@playwright/test';
type Kind = "from" | "to"

export class DateRangePicker{
  readonly page: Page  
  readonly calendar: FlatPickr
  readonly inputBoxFromDate: DateInput
  readonly inputBoxToDate:DateInput
  readonly me: Locator
  
  constructor(page:Page){    
    this.page = page    
    this.me = page.locator("cds-date-picker")    
    this.inputBoxFromDate = new DateInput(this.me, "from")
    this.inputBoxToDate = new DateInput(this.me, "to")
    this.calendar = new FlatPickr(this.me, page)
  }
  
  async closeCalendarEsc(){
    await this.page.keyboard.press('Escape')
  }  
    
  async closeCalendarBodyClick(){
    await this.page.locator('body').click()
  }
            
  private conv(t:string){ 
    const [y,m,d] = t.split("-").map(d=>parseInt(d,10)) 
    return Date.UTC(y,m-1,d )
  }
  
  async myRange(){
    // const dates = await this.me.evaluate( (el:any)=>el.calendar.selectedDates)
    const val = await this.me.evaluate((el:any)=>el.value)
    if (!val) return []
    
    const dates = val.split("/").map((d:string)=>new Date(this.conv(d)))
    return dates.length==1? [dates[0], undefined]: dates
  }
  
  async isCalendarDisplayed(){
    return await this.me.evaluate( (el:any)=> el.calendar.isOpen)
  }

  async getDisplayedMonthYear(){
    return await this.me.evaluate( (el:any)=> ({
      month: el.calendar.currentMonth,
      year: el.calendar.currentYear
    }))  
  }

  async myState(){
    const calendarMonth = await this.getDisplayedMonthYear()
    const myRange  = await this.myRange()
    const calendarDisplayed = await this.isCalendarDisplayed()
    return {calendarMonth, calendarDisplayed, myRange}
  }
}

```

1. The date range picker just initialises the date-inputs and the calendar. 
2. The only command/user actions on this class are to close the calendar. (Not sure if this is the right place, but for now it does not really matter). All other user actions are either on the inputsboxes or the flatpickr calendar.  
3. There are methods to extract the state.  The class itself does not store any state relevant to the component, but only extracts it from the component/html. 
4. The value property is where the date range is stored according to the carbon date picker docs. Since I had some problems with timezones, there is some conversion to spit out a utc date that can probably be improved. 
5. Also there is a calendar [property](https://open-wc.org/guides/knowledge/attributes-and-properties) in the drp-component  not to be confused with the calendar flatpickrclass.

</details>

Now, while writing this part is probably time consuming (if you do it in a pom/com or otherwise), but it is important to note that this is all code that you would likely will implement if you write e2e or ui tests for these components. 



### The model
Next up: the model. It needs to be a simplified representation of the sut (the date range picker). This particular model is a simplification because it only focusses on the range that is selected and the displayed year month that is displayed, and only on actions that a user can do with a mouse. Also, the way it keeps track of the selected dates is pretty basic. But normally, how complicated the model is up to you and dependend on the bugs you want to find.

One of the things I did not end up modelling for simplicity sake is picking a day from the previous or next month on the displayed calendar (there is always part of the last week of the previous month, and the first week of the next month displayed on the calendar). Other things left out of the model (and hence testing) are keyboard inputs, resetting the date picker, output like if the right range is higlighted, if today is correctly highlighted, if the right weekdays are displayed (and a lot more).

<details><summary markdown='span'>the model</summary>

  ```typescript
  export class DateRangeModel{
  isOpen: boolean =false
  monthCount:number
  startDate?:Date
  endDate?: Date
  openedInput: "from" | "to" | undefined
  private startCount
  

  private numToMY(num:number){
    const year = Math.floor(num /12)
    const month = num % 12
    return {year,month}
  }
  
  private dateToNum(date:Date){  
    const y = date.getFullYear()
    const m = date.getMonth()
    return y*12 + m
  }

  constructor(){
    console.log("new model")
    this.monthCount = this.dateToNum(new Date())
    this.startCount = this.monthCount
  }
  
  closeWithoutSelectingDate(){
    this.isOpen = false
    this.openedInput = undefined
    //possible bug replicated:  startdate not saved after closing if you do not also select an enddate 
    if (this.endDate == undefined){
      this.startDate = undefined
    }
  }
  
  openFromDate(){ 
    this.isOpen = true
    this.openedInput = "from"
    this.monthCount = this.startDate? this.dateToNum(this.startDate): this.startCount
    return this.numToMY(this.monthCount)
  }
  
  openToDate(){ 
    this.isOpen = true
    this.openedInput = "to"
    this.monthCount = this.endDate? this.dateToNum(this.endDate): this.startCount
    return this.numToMY(this.monthCount)
  }
 

  selectDay(day:number){
    const {year,month}= this.numToMY(this.monthCount)
    const dateChosen = new Date(Date.UTC(year,month,day))
    
    if (!this.startDate){//first date: always startdate, calendar stays open
      this.startDate = dateChosen
    }else if (!this.endDate){//second date: min=start max=end calendar closes
      if (this.startDate < dateChosen){
        this.endDate = dateChosen
      }else{
        this.endDate = this.startDate
        this.startDate = dateChosen
      }
      this.isOpen = false
      this.openedInput = undefined
    }else{//range update

      if (this.openedInput == "from"){
        if (dateChosen > this.endDate){
          //when does it swap dates when new date out of range??
          this.startDate = this.endDate
          this.endDate = dateChosen
        }else{          
          this.startDate =dateChosen
        }
      }else if(this.openedInput =="to"){
        if (dateChosen < this.startDate){
          this.endDate = this.startDate
          this.startDate = dateChosen 
        }else{
          this.endDate=dateChosen
        }
      }
      this.openedInput = undefined
      this.isOpen=false
    }
  }

  myDate(){
    return this.numToMY(this.monthCount)
  }
  
  selectNextMonth(){
    this.monthCount +=1
  }
  
  selectPreviousMonth(){
    this.monthCount -=1
  }

  selectNextYear(){
    this.monthCount +=12
  }

  selectPreviousYear(){
    this.monthCount -=12
  }

  hasRange(){
    return this.startDate && this.endDate
  }

  myState(){
    const calendarMonth = this.myDate()
    const myRange  =  [this.startDate, this.endDate]
    const calendarDisplayed = this.isOpen
    const openedInput = this.openedInput
    return {calendarMonth, calendarDisplayed, myRange, openedInput}
  }
}

  ```  
  
- the date is just stored as number (year *12 + month-1), that is parsed back to year month when needed. Increasing the date month just increases this number. 
- the logic to fill the dates (selectDay) required some reverse engineering the carbon date picker. But will see some problems here later on.
- The model might look more complicated then the coms logic wise. But that because the model simplifies the date picker component, not our datepicker class. Because the coms do not have any state (the carbon component does) it is a pretty straightforward.

</details>


### Fast-check setup
Fastcheck needs to know which commands are available and how to run the commands in both the sut and the model. 
So first we must define how fastcheck can execute the commands in both using fast-checks ICommand interface. 
Second, we need to turn all the commands together into what is called an arbitrary (The arbitrary is a fc term for a function that will generate random output of certain types). This is also where some commands get extra input that is randomised (for instance when choosing a day on the calendar).


#### Defening a command with the ICommand interface
 We need to implement commands in a class that conforms to the [ICommand interface](https://fast-check.dev/docs/advanced/model-based-testing/#write-model-based-testsinterface). That means implementing the following three functions:
-  *check*:  used to check if the command is possible to execute given the current state.  The check function takes in as parameter the model. 
-   *run*: used to execute the command for real (in the sut) and in the model. It is also were you do the assertions to check for errors. The run function has the real and the model as parameters.
-  *toString*: used to output a description of the commands when a failure occurs. Here is where you provide a name for the command (plus possible extras) to help with debugging. 

<details>
  <summary markdown='span'>
    An example of ICommand implementation for some action/command
  </summary>
  
```typescript
export class SelectDay implements Command<DateRangeModel, DateRangePicker>{
  day: number

  constructor(day: number) {
    this.day = day
  }

  check(m: Readonly<DateRangeModel>): boolean {
    return m.isOpen
  }

  async run(m: DateRangeModel, r: DateRangePicker) {
    await r.calendar.selectDay(this.day.toString())
    m.selectDay(this.day)
    await checkState(m, r)
  }

  toString(): string {
    return `picking day ${this.day} on the calendar`
  }

}
```
- The class implements the Command interface that also needs the model and the real class as type parameters.  
- The class implementing a command must have a constructor if the command needs random arguments from fastcheck that will be used when the command runs.  The SelectDay command needs extra input (day: number) and we add it to the state so it can be used in other functions.  We will see later on where this extra parameter is coming from. 
- The SelectDay command will only be run if the the calendar is open, as you need to define in the check function 
- The run function just does the relevant action in the model and the sut (dp) and then calls the checkState function (implementations not shows here) that does the assertions. 


</details>

<details>   
  <summary markdown='span'>
    All the commands defined
  </summary>
  


```typescript
import { Command } from "fast-check"

import { DateRangePicker } from "../com/actions"
import * as assert from "node:assert"
import { DateRangeModel } from "./model"

function log(method: any, context: any) {
  return async function (this: any, ...args: any[]) {
    console.log(this.toString())
    return await method.call(this, ...args)
  }
}

function logStateAfter(ison = false) {
  return function logState(method: any, context: any) {
    return async function (this: any, ...args: any[]) {
      const res = await method.call(this, ...args)

      if (ison) {
        console.log("model", args[0].myState())
        console.log("datepicker", await args[1].myState())
      }

      return res
    }
  }
}

function logStateAfterFailue(method: any, context: any) {
  return async function (this: any, ...args: any[]) {
    const modelBefore = args[0].myState()
    const realBefore = await args[1].myState()

    try {
      const res = await method.call(this, ...args)
      return res
    } catch (error) {
      // console.log(error)
      console.log("modelBefore", modelBefore)
      console.log("datepicker before", realBefore)
      console.log("modelAfter", args[0].myState())
      console.log("datepicker after", await args[1].myState())
      throw (error)
    }
  }
}

function sleep(ms: number) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

function pause(ms: number) {
  return function (method: any, context: any) {
    return async function (this: any, ...args: any[]) {
      await sleep(ms*0.6)
      return method.call(this, ...args)
    }
  }
}

const alwaysLogState = true

export class OpenCalendarFrom implements Command<DateRangeModel, DateRangePicker>{

  check(m: Readonly<DateRangeModel>) {
    return !(m.isOpen)
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, dp: DateRangePicker) {
    await dp.inputBoxFromDate.clickInputBox() //openFromDateInput()
    m.openFromDate()
    await dp.isCalendarDisplayed()
    await checkState(m, dp)
  }

  toString() {
    return "opening calendar using the 'from' input"
  }
}

export class OpenCalendarTo implements Command<DateRangeModel, DateRangePicker>{

  check(m: Readonly<DateRangeModel>) {
    return !(m.isOpen)
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, dp: DateRangePicker) {

    await dp.inputBoxToDate.clickInputBox()
    m.openToDate()
    await checkState(m, dp)

  }
  toString() {
    return "opening calendar using the 'to' input"
  }
}

export class CloseCalendarEsc implements Command<DateRangeModel, DateRangePicker>{

  check(m: Readonly<DateRangeModel>) {
    return m.isOpen
  }
  @log @logStateAfter(alwaysLogState) @logStateAfterFailue 
  async run(m: DateRangeModel, dp: DateRangePicker) {
    await dp.closeCalendarEsc()
    m.closeWithoutSelectingDate()
    await checkState(m, dp)
  }
  toString() {
    return "closing calendar by pressing escape"
  }
}

export class CloseCalendarClick implements Command<DateRangeModel, DateRangePicker>{

  check(m: Readonly<DateRangeModel>) {
    return m.isOpen 
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, dp: DateRangePicker) {
    await dp.closeCalendarBodyClick()
    m.closeWithoutSelectingDate()
    await checkState(m, dp)
  }
  
  toString() {
    return "closing calendar by clicking body"
  }
}

export class SelectDay implements Command<DateRangeModel, DateRangePicker>{
  day: number

  constructor(day: number) {
    this.day = day
  }

  check(m: Readonly<DateRangeModel>): boolean {
    return m.isOpen
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, r: DateRangePicker) {
    await r.calendar.selectDay(this.day.toString())
    m.selectDay(this.day)
    await checkState(m, r)
  }

  toString(): string {
    return `picking day ${this.day} on the calendar`
  }
}

export class SelectNextMonth implements Command<DateRangeModel, DateRangePicker>{
  check(m: Readonly<DateRangeModel>): boolean {
    return m.isOpen
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, r: DateRangePicker) {
    m.selectNextMonth()
    await r.calendar.selectNextMonth()
    await checkState(m, r)
  }

  toString(): string {
    return "selecting next month on the calendar"
  }
}

export class SelectPrevMonth implements Command<DateRangeModel, DateRangePicker>{
  check(m: Readonly<DateRangeModel>): boolean {
    return m.isOpen
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, r: DateRangePicker) {
    m.selectPreviousMonth()
    await r.calendar.selectPrevMonth()
    await checkState(m, r)
  }
  toString(): string {
    return "selecting previous month on the calendar"
  }
}

export class SelectNextYear implements Command<DateRangeModel, DateRangePicker>{
  check(m: Readonly<DateRangeModel>): boolean {
    return m.isOpen
  }

  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, r: DateRangePicker) {
    m.selectNextYear()
    await r.calendar.selectNextYear()
    await checkState(m, r)
  }
  toString(): string {
    return "selecting next year on the calendar"
  }
}

export class SelectPrevYear implements Command<DateRangeModel, DateRangePicker>{
  check(m: Readonly<DateRangeModel>): boolean {
    return m.isOpen
  }
  @log @logStateAfter(alwaysLogState) @logStateAfterFailue
  async run(m: DateRangeModel, r: DateRangePicker) {
    m.selectPreviousYear()
    await r.calendar.selectPrevYear()
    await checkState(m, r)
  }
  toString(): string {
    return "selecting previous year on the calendar"
  }
}


const formatDate = (d: any) => d?.toISOString().slice(0, 10) + "  -   "

const checkState = async (m: DateRangeModel, r: DateRangePicker) => {

  const modelState = m.myState()
  const dpState = await r.myState()

  assert.equal(modelState.calendarDisplayed, dpState.calendarDisplayed)
  
  if (modelState.calendarDisplayed) {
    assert.equal(modelState.calendarMonth.month, dpState.calendarMonth.month)
    assert.equal(modelState.calendarMonth.year, dpState.calendarMonth.year)
  }

  if (m.hasRange()) {
    assert.equal(modelState.myRange[0]?.getTime(), dpState.myRange[0]?.getTime(),
      `startdate different ${modelState.myRange.map(formatDate)} ${dpState.myRange.map(formatDate)}`)
    assert.equal(modelState.myRange[1]?.getTime(), dpState.myRange[1]?.getTime(),
      `enddate different ${modelState.myRange.map(formatDate)} ${dpState.myRange.map(formatDate)}`)
  }
}

```

- While it looks like a lot of code, there is a lot of boilerplate. The actual implementations of the commands is simple.  

- The output of fastcheck during a run is kind off minimal, which can be unsettling at first because you won't know what is happening. So best to write your own loggers in some methods.  I learned about decorators in typescript so some methods are  wrapped these decorators that log the method being executed, log the state, or adds a pause. Read these blogs for more details about the decorators: [1](https://devblogs.microsoft.com/typescript/announcing-typescript-5-0/#decorators) / [2](https://angularexperts.io/blog/typescript-decorators). But no need for decorators, there is plenty of other places to add a logging statement. 


</details>


#### Commands functions
Now that we defined what fc must execute and assert to complete a command, we need to turn them into an arbitrary. The arbitrary is a fc term for a function that will generate random output, in this case a sequence of commands that will be executed in the mbt. Every call of the commands arbitrary will generate a new random sequence. You convert the commands to an arbitrary by using to the fc.commands function.

Some of the commands, like picking a day on the calendar require some more random input: the actual day to pick in this case. We can use default arbitraries from fc (it has arbitraries to generate numbers, strings json and many other types) and pass them along the the command using the map function. Otherwise (if no inputs required) the command needs to be wrapped as an fc.constant(). See the details for an example. 



<details>
  <summary markdown='span'>Defining the commands arbitrary</summary>

```typescript

export const dateRangePickerCommands = fc.commands([
  fc.constant(new OpenCalendarFrom()),
  fc.constant(new OpenCalendarTo()),
  //fc.constant(new CloseCalendarEsc),
  fc.constant(new CloseCalendarClick()),
  fc.constant(new SelectNextMonth()),
  fc.constant(new SelectPrevMonth()),
  fc.constant(new SelectNextYear()),
  fc.constant(new SelectPrevYear()),
  fc.integer({ min: 1, max: 28 }).map(d => new SelectDay(d)),
], { size: "xlarge" })

```

- the selectDay function needs a number that will be the actual day that is clicked on the currently displayed month. For simplicity, the max is 28 so it will not fail with the day not being present if a month with less days is chosen. 
- One thing to be aware of is that by default, not a lot of commands will be run. It will do a handfull (sometimes even zero), and then start over. If you want the model to run longer, you need to play around with the size parameter. the xlarge size will prefer longer simulation. 

</details>


### All together now: setting up for a run

Mbt in fastcheck follows the same setup as a regular property based test. You can read about how fast-check uses runners, properties and arbitraries in the [fastcheck docs](https://fast-check.dev/docs/core-blocks/) or [here]({% link _posts/2023-11-14-learning-property-based-testing.markdown %})). 

Short version:we need to build a property function function that takes in the arbitrary we defined above (dateRangePickerCommands) and a predicate function. Normally the predicate function has the arbitraries as params and returns true or false, but in the case of mbt it takes in the arbitrary and calls another fastcheck function: the model run. This modelrun function also needs to know how to initialise the sut and the model, so we need to provide a function for that. So this modelrun functions glues together the sequence of commands and the model and sut. 




Check out the details for an example:

<details><summary markdown='span'>Defining a model based test</summary>

```typescript
test('DateInputRange test', async ({ page }) => {
  
  const init = async () => {
    const real = new DateRangePicker(page)
    await page.goto("file:///home/joost/repos/carbon-date-picker/datepicker.html")
    return { model: new DateRangeModel(), real }
  }

  await fc.assert(
    fc.asyncProperty(
      dateRangePickerCommands,
      async (cmds) => {
        await fc.asyncModelRun(init, cmds);
      }
    ), { numRuns: 1, endOnFailure: false } //
  );
})
```
- in the init function a new instance is generated of the sut and the model and the playwright page fixture used by the daterangepucker com is send to the right url. 
- the daterangepickerCommands is our arbitrary, that will generate a random sequence of commands. 
- such a random sequence (cmds) is then passed allong to the modelrun function
- since playwright locators are asynchronous, we need to use the asyncronous versions of several fc functions. 
-Like in pbt fc for mbt will just run 100 tests by default. If an asserting fails at some point fastcheck will start shrinking to the most simple combinations of commands that trigger the error. 


</details>


<details>

- since this is using playwright test, you will need to increase the timeout in the config. 
- you can also use just playwright (without the tests) and setup your own page so no need to worry about timeouts, but that does not have the video / trace etc. 



</details>

Since we are using playwright, we can record what is happening (and use the trace to debug ). Here is an underwhelming screen recording of what is happening (with waits in between commands):

<video src="{{ site.baseurl }}/media/mbt/vid1.webm" style="max-width:600px;" controls preload > </video>>


### Does it find bugs and how is the output ?

Not sure if bugs, but there is definitely inconsistent behavior. This is just as an exampe of the types of bugs you might find (though other forms of testing would also find at least some of them)

Lets look at one of these examples, that also shows the importance of logging. 

<details>
  <summary markdown='span'>Before shrinking</summary>

- 0). This part is from the custom loggers. It shows that  2022-11-25 to 2024-01-08 was selected as date range. Then day 4 was selected with the calendar displaying sept 2021. The date picker was openend from the start date ("from" input). 

- 1). Mbt finds an issue. although the start date has been updated correctly, in the drp the end date is has also changed (to what the startdate was). It reports the sequence of actions that were taken, and a seed to replicate these actions. 

- Analysing this inconsistency would get very detailed. But it has to do with the drp sometimes swapping dates when a selected date is outside of the current range, in a way that makes no sense to me. This does not always happen, but has to do with the actions and state before hand. So it is a great example for model based testing. 

- Also important to note that I have excluded some commands from the fc.commands (closing the date picker, opening from "to input")

```bash

(logs left out)

0)

selecting previous year on the calendar
model {
  calendarMonth: { year: 2021, month: 8 },
  calendarDisplayed: true,
  openedInput: 'from'
}
datepicker {
  calendarMonth: { month: 8, year: 2021 },
  calendarDisplayed: true,
  myRange: [ 2022-11-25T00:00:00.000Z, 2024-01-08T00:00:00.000Z ]
}
picking day 4 on the calendar
modelBefore {
  calendarMonth: { year: 2021, month: 8 },
  calendarDisplayed: true,
  myRange: [ 2022-11-25T00:00:00.000Z, 2024-01-08T00:00:00.000Z ],
  openedInput: 'from'
}
datepicker before {
  calendarMonth: { month: 8, year: 2021 },
  calendarDisplayed: true,
  myRange: [ 2022-11-25T00:00:00.000Z, 2024-01-08T00:00:00.000Z ]
}
modelAfter {
  calendarMonth: { year: 2021, month: 8 },
  calendarDisplayed: false,
  myRange: [ 2021-09-04T00:00:00.000Z, 2024-01-08T00:00:00.000Z ],
  openedInput: undefined
}
datepicker after {
  calendarMonth: { month: 8, year: 2021 },
  calendarDisplayed: false,
  myRange: [ 2021-09-04T00:00:00.000Z, 2022-11-25T00:00:00.000Z ]
}


  1) tests/model.spec.ts:32:5 › DateInput com ──────────────────────────────────────────────────────

    Error: Property failed after 1 tests
    { seed: 1799548799, path: "0", endOnFailure: true }
    Counterexample: [opening calendar using the 'from' input,selecting previous year on the calendar,picking day 25 on the calendar,selecting next year on the calendar,selecting previous year on the calendar,picking day 17 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,selecting previous month on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting next month on the calendar,selecting previous year on the calendar,picking day 2 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting next year on the calendar,picking day 15 on the calendar,opening calendar using the 'from' input,selecting next month on the calendar,selecting next year on the calendar,selecting previous month on the calendar,selecting next year on the calendar,selecting next year on the calendar,selecting next month on the calendar,picking day 23 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,selecting previous year on the calendar,selecting next year on the calendar,picking day 25 on the calendar,opening calendar using the 'from' input,selecting next year on the calendar,selecting previous year on the calendar,picking day 26 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,selecting next year on the calendar,selecting previous month on the calendar,picking day 9 on the calendar,opening calendar using the 'from' input,picking day 21 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,selecting next year on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting previous month on the calendar,picking day 15 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,selecting previous year on the calendar,picking day 22 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,selecting previous month on the calendar,selecting previous month on the calendar,selecting next year on the calendar,selecting next year on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting next month on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting next month on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting previous month on the calendar,selecting next month on the calendar,picking day 14 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,picking day 21 on the calendar,opening calendar using the 'from' input,selecting next month on the calendar,selecting previous month on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,picking day 22 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,picking day 5 on the calendar,opening calendar using the 'from' input,picking day 1 on the calendar,opening calendar using the 'from' input,selecting next month on the calendar,picking day 28 on the calendar,opening calendar using the 'from' input,selecting next year on the calendar,selecting next year on the calendar,picking day 17 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,picking day 10 on the calendar,opening calendar using the 'from' input,picking day 19 on the calendar,opening calendar using the 'from' input,picking day 19 on the calendar,opening calendar using the 'from' input,selecting next year on the calendar,selecting next month on the calendar,selecting next year on the calendar,picking day 16 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,selecting previous month on the calendar,selecting next year on the calendar,selecting next year on the calendar,selecting previous year on the calendar,picking day 1 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,picking day 26 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,selecting next year on the calendar,selecting next month on the calendar,picking day 15 on the calendar,opening calendar using the 'from' input,picking day 18 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,selecting next year on the calendar,selecting next month on the calendar,selecting next month on the calendar,selecting previous month on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting previous month on the calendar,selecting next year on the calendar,selecting next month on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,picking day 27 on the calendar,opening calendar using the 'from' input,selecting next month on the calendar,selecting previous month on the calendar,selecting next year on the calendar,selecting previous year on the calendar,selecting next year on the calendar,selecting previous month on the calendar,selecting next year on the calendar,picking day 1 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,picking day 10 on the calendar,opening calendar using the 'from' input,selecting previous year on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting next year on the calendar,selecting next year on the calendar,picking day 2 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,picking day 17 on the calendar,opening calendar using the 'from' input,selecting next year on the calendar,selecting next year on the calendar,selecting next year on the calendar,picking day 8 on the calendar,opening calendar using the 'from' input,selecting previous month on the calendar,selecting next year on the calendar,selecting previous year on the calendar,selecting previous month on the calendar,selecting previous year on the calendar,selecting next year on the calendar,selecting previous year on the calendar,picking day 4 on the calendar /*replayPath=":"*/]
    Shrunk 0 time(s)
    Got AssertionError [ERR_ASSERTION]: enddate different 2021-09-04  -   ,2024-01-08  -    2021-09-04  -   ,2022-11-25  -   

```
</details>


<details>
  <summary markdown='span'>After shrinking</summary>


This is how fc reports about the inconsistency after shrinking. 

```bash
  1) tests/model.spec.ts:32:5 › DateInput com ──────────────────────────────────────────────────────

    Error: Property failed after 1 tests
    { seed: 1799548799, path: "0:2:11:16:20:17:13:15:14:14:14:15", endOnFailure: true }
    Counterexample: [opening calendar using the 'from' input,picking day 1 on the calendar,picking day 1 on the calendar,opening calendar using the 'from' input,picking day 2 on the calendar,opening calendar using the 'from' input,picking day 1 on the calendar /*replayPath="DFCCADAAMFACGDADLEODBBACEBDEBBAAACJAACALABFAABCEAAECCBFKBAIAAABEDFADPGSAACACAFAAAACAAEACACFFBABCFAADBBAG/////////////////////////////////////////////////////////////////////////////////////////////////////////LBAI/M/M:qqqqqqqqqqqqqqqqqCAAAAAAAAAAAAAAAAAlB"*/]
    Shrunk 11 time(s)
    Got AssertionError [ERR_ASSERTION]: enddate different 2023-11-01  -   ,2023-11-02  -    2023-11-01  -   ,2023-11-01  - 
```

- So before the bug, the drp is set to  from 2023-11-01 to 2023-11-02, then opening the from input and selecting day 1 will set the date range to 2023-11-01 - 2023-11-01. Which looks like a bug to me related to when it makes sense to swap start and end dates if the newly selected date is outside of the current range.  
- Shrinking has reduced the number of commands to 7 (the original issue was 15 times that). This minimal example would be a great starting point for further debugging the issue. 



</details>

Other inconsistencies: 

- just selecting a startdate (and no end-date) and then closing the calendar will reset the whole calendar. Meaning that opening the calendar again, does not show the earlier selected startdate. (though not sure if this is intentional or not)

- first move the calendar to a different month or year and then close it without selecting a date: if you now open the from input, the calendar is reset to the current month, if you open the from date, the calendar is where you left it. 

- I am pretty sure that if you add commands for typing to it, there are more bugs to be found. 


## Thoughts about mbt

Please remember, just some thoughts based on this one simple example:

- Overall a very fun way to test.. 

- Lots of the playwright code (the component object models) would already be implemented if you had written some e2e or ui tests. Also no need to do in as a com/pom, though for me it makes sense. 

- The initial setup of the code with fastcheck is pretty straightforward if you are familiar with classes and does not take too much time. The way it is set up in fastcheck is really nice: Everything has a logical place which guides your coding.  (though you probably need an afternoon to get confortable with all the parts)

- Once you have a basic model running, adding new commands is easy. 

- Was expecting some flaky stuff from playwright and was questioning whether I should use a jsdom like solution, but have not encountered any issues with this. If your page / component loads quickly, does not stall, delay, suddenly rerender,  playwright will do great with this form of testing components. 

- Also shrinking (although pretty slow in this example) is a really nice way to find the simplest sequence of commands that trigger an inconsistency. Though on some runs it seemed the problem it shrunk to was not the same as the first one it encountered (but that is probably because there are multiple issues with the model/ bugs)

- If you want to be sure the test passes along some state, some tweaking / combining of the commands or just temporarily removing some commands might make sense, instead of using the commands in its plainest form. For instance, it sometimes takes a number of commands before a range is selected (because closing the calendar is also a command). Maybe combining actions (like for instance selecting a start and end date) for some runs is more efficient.  

- I feel pretty confident that building a mbt for somewhat complex components  would pay off. Also because a lot of the code can be re-used. 

- Would be interesting to see where it gets too complex for model based testing: could you build a test for some dashboad, or some mail app ? Or would too many possible commands just get forever more confusing  


## tips / general approach
- make the simplest model first, should be build gradually, start with the smallest possible, add one command at a time. 

- I would recommand write your own logging statement and to spend time to log as nicely as possible because debugging can be confusing and the default fc output can leave you guessing what it is doing. 

- Even with decent logging statements, things will get confusing at some point. Don't forget that the model will probably be wrong more often then the sut. 

- You might want to disable shrinking during developing your tests because it is not quick with mbt in this example, and lots of errors are also easily understood without shrinking if you have some nice logging. 

- Remember that mbt with fc does not test if actions that should be unavailable are actually impossible (unlike monkey testing). But you can always add an extra assertion that checks for instance if a button is disabled or something is not displayed in the ICommand check function .  

- Not everything needs to be modelled. If something that is not modelled is important to the state, you can always just update the model with info from the sut after executing a command in the com for the sut (not shown in this example, but usefull nontheless)
