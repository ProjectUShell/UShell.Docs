# The 'mapDynamic'-Approach (piping of UOW states)



A 'mapDynamic' structure can be placed beside of any argument-properties. This structure will never be passed as argument to the corresponding Widget/UseCase/Provider! Instead, It will be evaluated, and the result will be passed as Argument like declared in "for"!



**"for"**: defines the target argument to overwrite

**"use"** defines the source, where a dynamical value is taken from. For that there are multiple providers:

| channel               | provides values from...                                      |
| --------------------- | ------------------------------------------------------------ |
| "setting://"          | ... the wellknown 'settings'-datasource                      |
| "unitOfWork://"       | ... the runtime-state of that (source-)usecase, from which the current command was triggered |
| "applicationScope://" | ... the portfolio.json file, out of the section named "applicationScope" |
| "userInput://"        | ... a modal overlay dialog, which queries the user to provide a value |

In the example below

* the "url" argument will be provided from the setting called "BackendUrl" from the section named "BackendConfiguration"

* a "tenant" identifier will be loaded out of the portfoliojson and mapped so that the default value '*' of 'dataScope.tenant' will be overwritten by that value.

  

```json
initUnitOfWork:{
   url: '',
   selectedItemIdentity: null,
   defaultFilter:{
       stichtag: null;
   }
   dataScope:{
       tenant: '*'
   },
   mapDynamic:[
      {use: "setting://BackendConfiguration.BackendUrl", for: "url"}
      {use: "useCaseState://selectedItem.Id", for: "selectedItemIdentity"}
      {use: "applicationScope://tenant", for: "dataScope.tenant"}
      {use: "userInput://Stichtag", for: "defaultFilter.stichtag"}
   ]
},


```


