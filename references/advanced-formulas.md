# Advanced Formulas Reference

**Source**: SAP Analytics Cloud: Advanced Formulas Reference Guide (2025.19)
**URL**: https://help.sap.com/doc/5516580733124039b673c530125771b3/2025.19/en-US/index.html
**Last updated in this file**: 2026-10-18
**Scope**: Advanced formula steps in SAC Data Actions — syntax, functions, operators, examples

> This file is extracted from official SAP documentation and reformatted for use with the Claude skill.
> Update quarterly when a new QRC is released. Check the SAP Help URL above for changes.

---

Use this reference guide to learn about the syntax and semantics of advanced formula steps in SAP Analytics Cloud data actions. It's meant for administrators with an understanding of planning processes and models within SAP Analytics Cloud. You don't need previous scripting experience, though. Advanced formula steps use their own scripting language, which is similar to formulas in SAP Analytics Cloud models. If you write SQL script, you'll also recognize some instructions. 
---

## Introduction to Advanced Formulas 

Advanced formula steps in data actions give you a lot more flexibility to run calculations on planning model data. The calculation results are stored in the base cells of the model, which means that you can run calculations and copy data across different sets of filters. Similar syntax is available when creating calculated accounts and measures, but most of that syntax is function-based. Planning with advanced formulas requires more procedural functionality, so it exists as a separate language. 
---

## Notation Convention 

This reference uses syntax notation very similar to Backus-Naur Form (BNF), a notation technique used to define programming languages. 
---

## Symbols 

This reference guide uses the following symbols to show the relationships between syntax terms. 
| Symbol  || Description  |
| < >  || Angle brackets surround the name of a syntactic element.  |
| ::=  || The definition operator defines the element that appears to its left.  |
| []  || Square brackets indicate optional elements in a formula. You can choose whether to specify or omit optional elements in your script.  |
| {}  || Braces group elements in a formula. Repeating elements (zero or more elements) can be specified within brace symbols.  |
| |  || The alternative operator indicates that the portion of the formula following the bar is an alternative to the portion preceding it.  |
| [...]  || An ellipsis with square brackets around it indicates optional repetition of the	preceding element or grouped elements. 
For example, if you can specify one or more columns for an option and must separate them by commas, this is expressed as: <column_name> [,...]. 
An example of grouped elements that don't need a comma separator looks like this: { <column_name> <data_type> } [...]  |
---

## Syntax Term Representations 

Throughout this reference guide, each syntax term is defined using one of the following representations: 
```
<model> ::= "Model_Name"
<linked_model> ::= Linked<model>
<version_dim> ::= [d/Version_Dimension_Name]
<measure_dim>::= [d/Measures]
<account_dim> ::= [d/Account_Dimension_Name]
<date_dim> ::= [d/Date_Dimension_Name]
<generic_dim> ::= [d/Generic_Dimension_Name]
<organization_dim> ::= [d/Organization_Dimension_Name]
<dimension> ::= [<account_dim> | <date_dim> | <generic_dim> | <organization_dim>]
<measures/dimension> ::= [<measure_dim> | <account_dim> | <time_dim> | <generic_dim> | <organization_dim>]
<property> ::= [p/Property_ID]
<hierarchy> ::= [h/Hierarchy_ID]
<member> ::= [<leaf_member> | <parent_member>]
<parent_member> ::= "Parent_Member_ID"
<parent_members> ::= <parent_member> [, ...]
<leaf_member> ::= "Leaf_Member_ID"
<leaf_members> ::= <leaf_member> [, ...]
<measure>::="Measure_Member"
<leaf_member/measure> ::= [<leaf_member>|<measure>]
<leaf_members/measures> ::= <leaf_member/measure> [, ...]
<date_hierarchy> ::= <date_dim>.<hierarchy>
<date_member> ::= <leaf_member> of <date_dim>
<property_value> ::= "Value" of <property>
<#variablemember> ::= #VariableMember
<@integer> ::= "INTEGER Variable"
<@float> ::= "FLOAT Variable"
<number_variable>::= {<@integer> | <@float>}
<parameter_any_setHier_leaf> ::= %Parameter% (Type: Member/Measure; Cardinality: Any; Hierarchy: Set Hierarchy; Level: Leaf)
<parameter_single_setHier_leaf> ::= %Parameter% (Type: Member/Measure; Cardinality: Single; Hierarchy: Set Hierarchy; Level: Leaf)
<parameter_single_setHier_any> ::= %Parameter% (Type: Member/Measure; Cardinality: Single; Hierarchy: Set Hierarchy; Level: Any)
<parameter_any_setHier_any> ::= %Parameter% (Type: Member/Measure; Cardinality: Any; Hierarchy: Set Hierarchy; Level: Any)
<parameter_any_anyHier_leaf> ::= %Parameter% (Type: Member/Measure; Cardinality: Any; Hierarchy: Any; Level: Leaf)
<parameter_single_anyHier_leaf> ::= %Parameter% (Type: Member/Measure; Cardinality: Single; Hierarchy: Any; Level: Leaf)
<parameter_single_anyHier_any> ::= %Parameter% (Type: Member/Measure; Cardinality: Single; Hierarchy: Any; Level: Any)
<parameter_any_anyHier_any> ::= %Parameter% (Type: Member/Measure; Cardinality: Any; Hierarchy: Any; Level: Any)
<parameter_number> ::= %Parameter% (Type: Number)

<operator> ::= + | - | / | *
<comparator> ::= != | = | !== | ==
<comparator_all> ::= < | > | = | <= | >= | !=
<and_or> ::= AND | OR
<digit> ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
<sign> ::= + | -
<period> ::= .
<unsigned_integer> ::= <digit>[...]
<signed_integer> ::= [<sign>] <unsigned_integer>
<unsigned_numeric> ::= <unsigned_integer> [ <period> [ <unsigned_integer> ] ] | <period> <unsigned_integer>
<signed_numeric> ::= [<sign>] <unsigned_numeric>
<numeric> ::= <signed_numeric> | <signed_integer>

<fixed_date> ::= YYYY-MM-DD | YYYY-MM | YYYY-Q | YYYY | YYYYMMDD | YYYYMM | YYYYQ | YYYY/MM/DD | YYYY/MM | YYYY/Q | YYYY.MM.DD | YYYY.MM | YYYY.Q

```

# Global Configuration Functions 

Global configuration functions let you set global configuration options that are applied to the entire advanced formulas step. 
---

## CONFIG.TIME_HIERARCHY 

This configuration governs the time hierarchy of the script.
If this configuration is not defined, the default time hierarchy from the model is used. 
### Syntax 

```
CONFIG.TIME_HIERARCHY = {CALENDARYEAR | FISCALYEAR}

```

---

## CONFIG.GENERATE_UNBOOKED_DATA 

This option specifies how to deal with unbooked source data. Unbooked data refers to cells with empty values. 
Turn it on to treat unbooked source cells as 0 value. In this case, unbooked cells can be included in data to copy and change target value to 0.
If turned off (the default setting), only booked data will be copied. Unbooked data in the calculation results will be ignored and won't change target values. 
### Syntax 

```
CONFIG.GENERATE_UNBOOKED_DATA = {ON | OFF}

```

---

## CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE 

If this configuration is turned on, all calculations consider the sign value of debit accounts (expense or asset) or credit accounts (income or liability and equity), as shown in the model's fact table. When it's set to off (the default setting), RESULTLOOKUP calculations return account values with automatic sign switching applied to INC and LEQ accounts. In this case, account values will usually be positive numbers. 
### Syntax 

```
CONFIG.FLIPPING_SIGN_ACCORDING_ACCTYPE = {ON | OFF}

```

---

## CONFIG.HIERARCHY 

This configuration governs the hierarchy of one or more dimensions you will be working on in a script. For any dimension that you don't include, the default hierarchy of the dimension will be used instead. This configuration can also be used to govern the hierarchy of a linked model. When specifying the hierarchy of a dimension in a linked model, you can either use CONFIG.HIERARCHY outside a MODEL statement or HIERARCHY inside one. 
### Syntax 

```
CONFIG.HIERARCHY [<linked_model> =]{<dimension>.<hierarchy>}[,...]

```

```
MODEL < linked_model >
    HIERARCHY { < dimension > . < HIERARCHY > } [, ... ]
ENDMODEL

```

---

## CONFIG.TIME_ZONE_OFFSET 

This configuration redefines UTC base time with the time zone offset by country for function such as TODAY(). 
### Syntax 

```
CONFIG.TIME_ZONE_OFFSET = {-23 | -22 | ... | +22 | +23}

```

---

## CONFIG.READ_CALCULATED_MEMBER_VALUES 

This option defines if the values of calculated members, which include non-leaf members, account members with formulas, and calculated measures, can be read in the advanced formulas script. If turned ON, you can define calculated members in RESULTLOOKUP and LINK functions. If turned OFF (default setting), you will run into validation errors if any calculated member is defined in RESULTLOOKUP or LINK function. This configuration function is only supported on SAP Analytic Cloud tenants based on SAP HANA Cloud platform. 
### Syntax 

```
CONFIG.READ_CALCULATED_MEMBER_VALUES = {ON | OFF}

```

---

## CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY 

This configuration expands the calculation scope to include members that aren't in the hierarchy for one or more dimensions. This configuration can also be used to include not in hierarchy member of a linked model. When specifying the dimension in a linked model, you can either use CONFIG.HIERARCHY.INCLUDE MEMBERS NOT IN HIERARCHY outside a MODEL statement or HIERARCHY.INCLUDE MEMBERS NOT IN HIERARCHY inside one. 
### Syntax 

```
CONFIG.HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY [<linked_model>=]{<dimension>}[,...]

```

```
MODEL < linked_model >
    HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY {<dimension>}[,...]
ENDMODEL

```

# Member Selector Functions 

Member selector functions help set the overall scope of your step or return members of a dimension that match certain criteria. 
---

## MEMBERSET 

Use this statement to define the overall dimension member scope.
MEMBERSET statements let you pick which members to include or exclude in your advanced formulas step. 
### Syntax 

```
MEMBERSET {<dimension_member>|<dimension_property>|<date_scope>}

```

### Syntax Elements 

```
<dimension_member> ::= {<measures/dimension> <comparator> <member_id>}
<member_id> ::= {(<leaf_member>[,...]) | <dimension >.<property> | (<parameter_list>[,...]) | (<basemember>[,...])}
<parameter_list> ::= <parameter_any_setHier_leaf> | <parameter_any_anyHier_leaf> | <parameter_single_setHier_leaf>| <parameter_single_anyHier_leaf>
<basemember> ::= BASEMEMBER()
<dimension_property> :: = {<dimension>.<property> <comparator> <property_values>[,...]}
<property_values> ::= <property_value> | <numeric_property> | <property_scope>
<property_scope> ::= <numeric_property> TO <numeric_property> 
<numeric_property> ::= <numeric> | <parameter_number>
<date_scope> ::= {<date_dim> <comparator> <date_member_id> TO <date_member_id>}
<date_member_id> ::= [<date_member> | <version_dim>.<property> | PERIOD () | NEXT() | PREVIOUS ()]

```

### Examples 

```
MEMBERSET [d/Flow] = ("F_TEMP", "F_NONE")

```

```
MEMBERSET [d/Currency] = [d/Organization].[p/Currency]

```

```
MEMBERSET [d/Entity].[p/Region] = "ASIA"

```

```
MEMBERSET [d/PRODUCT].[p/FACTOR] != 100

```

```
MEMBERSET [d/Account] = BASEMEMBER([d/Account], %Input_Account%)

```

```
MEMBERSET [d/PRODUCT] != BASEMEMBER( [d/PRODUCT].[h/H1], "All_Clothes")

```

```
MEMBERSET [d/Date] = "202001" TO "202012"

```

```
MEMBERSET [d/Date] != "202001" TO "202012"

```

```
MEMBERSET [d/Date] = [d/Version].[p/Beginning] TO [d/Version].[p/Ending]

```

```
MEMBERSET [d/Date] = [d/Version].[p/Beginning] TO PERIOD(TODAY())

```

```
MEMBERSET [d/Measures] = ("Amount", "Quantity", "Price")

```

```
MEMBERSET [d/Measures] != ("Amount", "Quantity", "Price")

```

```
MEMBERSET [d/PRODUCT].[p/FACTOR] = %numberParameter%

```

```
MEMBERSET [d/PRODUCT] != %memberParameter%

```

```
MEMBERSET [d/PRODUCT].[p/FACTOR] = 100 TO 300

```

```
MEMBERSET [d/PRODUCT].[p/FACTOR] = 100 TO %numberParameter%

```

---

## BASEMEMBER 

This function sets the scope to all base members of one or more specific parent members. Base members refer to the leaf members without any child members in a given hierarchy. 
### Syntax 

```
BASEMEMBER(<dimension_hierarchy>,{<parent_member_id>[,...]})

```

### Syntax Elements 

```
<dimension_hierarchy> ::= {<dimension> | <dimension>.<hierarchy>} 
<parent_member_id> ::= {<parent_member> | <parameter_single_setHier_any> | <parameter_single_anyHier_any> | <parameter_any_setHier_any>| <parameter_any_anyHier_any>}

```

### Examples 

```
CONFIG.HIERARCHY = [d/ENTITY].[h/H1]
MEMBERSET [d/ENTITY] = BASEMEMBER([d/ENTITY], "World")

```

```
MEMBERSET [d/ENTITY] = BASEMEMBER([d/ENTITY].[h/H2], "SalesCompany")

```

```
MEMBERSET [d/ENTITY] = BASEMEMBER([d/Account], %Account_Input%)

```

```
CONFIG.HIERARCHY = [d/ENTITY].[h/H1]
MEMBERSET [d/ENTITY] = BASEMEMBER([d/ENTITY], "EUROPE", "ASIA")

```

```
MEMBERSET [d/ENTITY] = BASEMEMBER([d/ENTITY], %Region1%, %Region2%)

```

```
MODEL [HR]
    MEMBERSET [d/Account_HR] = BASEMEMBER([d/Account_HR].[h/parentId],"Salary_Month_Total" )
ENDMODEL

```

---

## ELIMMEMBER 

This function returns the names of the elimination members below the first common parent of two companies that did inter-company trading. 
In the case above, the inter-company transaction data between Korea and China will be eliminated after copying to Elimination Asia, which is the elimination member under their common parent, Asia. For the inter-company transaction data between Korea and France, the value will be copied to Elimination World under their first common parent, World. Using ELIMMEMBER, you can get Elimination Asia as the elimination member of Korea and China and Elimination World as the elimination member of Korea and France. 
### Syntax 

```
ELIMMEMBER(<elim_dimension_hierarchy>,<elim_member1>,<elim_member2>,[<elim_property>="Y"])

```

### Syntax Elements 

```
<elim_dimension_hierarchy> ::= <elim_dimension>|<elim_dimension>.<hierarchy> 
<elim_dimension> ::= <dimension> 
<elim_member1> ::= <leaf_member> of <elim_dimension>|<elim_dimension> 
<elim_member2> ::= <leaf_member> of <elim_dimension>|<elim_dimension> 
<elim_property>::=<elim_dimension>.<property>

```

### Examples 

```
DATA([d/Entity] = ELIMMEMBER([d/Entity].[h/Mgmt], "Japan", "France", [d/Entity].[p/Elimination] = "Y")) = RESULTLOOKUP()

```

```
DATA([d/Entity] = ELIMMEMBER([d/Entity], "GreatBritain", "France", [d/Entity].[p/Elimination] = "Y")) = RESULTLOOKUP()

```

```
DATA([d/Entity] = ELIMMEMBER([d/Entity], [d/Entity], [d/INTCO].[p/Entity], [d/Entity].[p/Elimination] = "Y")) = RESULTLOOKUP()

```

```
DATA([d/Entity] = ELIMMEMBER([d/Entity], [d/Entity].[p/IntCo], [d/IntCo])) = RESULTLOOKUP()

```

# Member Aggregation Functions 

These functions let you aggregate the members of certain dimensions. 
---

## AGGREGATE_DIMENSIONS 

Use this function to define the list of dimensions from the default model that will be aggregated before performing any calculations. 
### Syntax 

```
AGGREGATE_DIMENSIONS = <dimension>[,...]

```

### Examples 

```
AGGREGATE_DIMENSIONS = [d/Region], [d/Product]
AGGREGATE_WRITETO [d/Region] = "HQ"
AGGREGATE_WRITETO [d/Product] = "#"

```

```
AGGREGATE_DIMENSIONS = [d/Product]
AGGREGATE_WRITETO [d/Product] = "#"
MEMBERSET [d/Product] = ("P1", "P2", "P3")

```

---

## AGGREGATE_WRITETO 

Use this function to define a leaf member for each aggregated dimension. This leaf member will receive any values that are written to the dimension using the DATA function. 
In effect, it works like the Aggregate To option for copy steps and cross-model copy steps in a data action. 
### Syntax 

```
AGGREGATE_WRITETO [<dimension>=<member_id>]

```

### Syntax Elements 

```
<member_id> ::= {<leaf_member>|<parameter_single_setHier_leaf>|<parameter_single_anyHier_leaf>}

```

### Examples 

```
AGGREGATE_DIMENSIONS = [d/Region], [d/Product]
AGGREGATE_WRITETO [d/Region] = "HQ"
AGGREGATE_WRITETO [d/Product] = "#"

```

```
AGGREGATE_DIMENSIONS = [d/Product]
AGGREGATE_WRITETO [d/Product] = "#"
MEMBERSET [d/Product] = ("P1", "P2", "P3")

```

---

## MODEL 

To aggregate dimensions of a linked model, define AGGREGATE_DIMENSIONS functions inside a MODEL statement. 
You can also use MEMBERSET functions within a MODEL statement to filter the aggregated dimensions. The MODEL statement can also be used to define a linked model scope. 
### Syntax 

```
MODEL <linked_model>
  AGGREGATE_DIMENSIONS 
  [MEMBERSET]
ENDMODEL

```

```
Define the linked MODEL scope

MODEL <linked_model>
  [Configuration Functions] 
  [MEMBERSET]
ENDMODEL

```

### Syntax Elements 

```
<linked_model> ::= Linked<model>

```

### Examples 

```
MODEL [BusinessPlanning_Ref]
    AGGREGATE_DIMENSIONS = [d/Customer], [d/Entity]
ENDMODEL

```

```
MODEL [BusinessPlanning_Sales]
    AGGREGATE_DIMENSIONS = [d/Channel], [d/StoreType], [d/Product_lineitem]
    MEMBERSET [d/Channel] = ("Distribution", "Online", "Direct")
ENDMODEL

```

```
MODEL [Sales]
    HIERARCHY.INCLUDE_MEMBERS_NOT_IN_HIERARCHY = [d/Product_Item]
    MEMBERSET [d/Channel_Sales] = ("Direc","Distribution","Online","Others","Retail")
ENDMODEL

```

```
MODEL [BusinessPlanning_Sales]
    AGGREGATE_DIMENSIONS = [d/Channel], [d/StoreType], [d/Product_LineItem]
    MEMBERSET [StoreType] != "Department_Store"
    MEMBERSET [d/Channel] = ("Distribution", "Online", "Direct")
    MEMBERSET [d/Product_LineItem] = BASEMEMBER([d/Product_LineItem].[h/H1], "All_Footwear")
ENDMODEL

```

# Variable Definition 

These statements let you define variables in your script. A variable is a named data object that refers to a number or virtual dimension member. 
---

## VARIABLEMEMBER 

You can define this virtual member to store intermediate calculated values that can be reused elsewhere, whenever necessary, to avoid calculating the value repeatedly. 

The virtual variable member will be stored independently and cannot be added to a dimension like a real dimension member. After defining a variable member for a certain dimension, you can use the variable member to store values for that specific dimension only. 
### Syntax 

```
VARIABLEMEMBER #VariableMemberName OF <version/measures/dimension>

```

### Examples 

```
VARIABLEMEMBER #SumOfSales OF [d/Account]

```

```
VARIABLEMEMBER #Planning OF [d/Version]

```

```
VARIABLEMEMBER #Total_Revenue OF [d/Measures]

```

---

## INTEGER 

Defines a variable for your script with an integer data type. The default value is 0. 
### Syntax 

```
INTEGER @IntegerVariableName

```

### Examples 

```
INTEGER @iCOUNT

FOREACH [d/Product]
    @iCOUNT = @iCOUNT + 1
ENDFOR

```

---

## FLOAT 

Defines a variable for your script with a floating-point data type. The default value is 0.0. 
### Syntax 

```
FLOAT @FloatVariableName

```

### Examples 

```
FLOAT @SumOfValue

@SumofValue = RESULTLOOKUP([d/ACCOUNT] = "Profit")

```

# Write Back Functions 

These functions let you write back or delete records where a specified condition is met. 
---

## DELETE 

This function lets you clear data facts within a certain data scope. 
### Syntax 

```
DELETE (<measures/dimension>=<member_id>[,...],[<date_dim>=<date_scope_function>])

```

### Syntax Elements 

```
<member_id> ::= <leaf_member> | <dimension >.<property> | <parameter_single_setHier_leaf>| <parameter_single_anyHier_leaf> 
<date_scope_function> ::= [ PERIOD () | NEXT() | PREVIOUS () | FIRST () | PREYEARLAST () ]

```

### Examples 

```
DELETE([d/Account] = "Disposal", [d/FLOW] = "OPEN")

```

```
DELETE([d/Entity] = "Singapore", [d/INTCO] = [d/Entity].[p/IntCo])

```

```
DELETE([d/Measures] = "Measure1")

```

---

## DATA 

This function lets you write values and create records based on the scope. By default, DATA functions overwrite existing values in the target scope with specific values, or with data filtered by the RESULTLOOKUP function. 
### Syntax 

```
DATA([<measures/dimension>=<member_id>,...],[<date_dim>=<date_scope_function>])

```

### Syntax Elements 

```
<member_id> ::= { <leaf_member> | <#VARIABLEMEMBER> | <parameter_single_setHier_leaf> |<parameter_single_anyHier_leaf> | <dimension> | [<linked_model>.]<dimension> | [<linked_model>.]<dimension>.<property> | ELIMMEMBER() } 
<date_scope_function> ::= [ PERIOD () | NEXT() | PREVIOUS () | FIRST () | PREYEARLAST () ]

```

### Examples 

```
DATA([d/Account] = "90011001")) = RESULTLOOKUP([d/Account] = "11101000") + RESULTLOOKUP([d/Account] = "22000000")

```

```
VARIABLEMEMBER #Account OF [d/Account]
DATA([d/Account] = #Account) = RESULTLOOKUP([d/Account] = "90011001") * 1.2

```

```
DATA([d/Account] = %InputAccount%) = RESULTLOOKUP([d/Account] = "90011001") * 1.2

```

```
DATA([d/Entity] = [d/CostCenter].[p/Region]) = RESULTLOOKUP([d/Entity] = "HQ_Europe")

```

```
DATA([d/Measures] = "Amount") = RESULTLOOKUP([d/Measures] = "Quantity") * RESULTLOOKUP([d/Measures] = "Price")

```

```
DATA([d/Date] = PREVIOUS()) = RESULTLOOKUP()

```

```
DATA([d/Date] = NEXT(12)) = RESULTLOOKUP()

```

```
DATA([d/Entity]) = ELIMMEMBER([d/Entity], "CHINA", "KOREA", [d/Entity].[p/Elimination] = "Y")) = RESULTLOOKUP()

```

```
MODEL [HR]
ENDMODEL

DATA([d/Measures] ="Closing", [d/Account_Fin] = [HR].[d/Account_HR].[p/GL_Acc]) = LINK([HR], [d/Measures] = "Salary_Amt", [d/Version] = "public.Plan01")

```

---

## DATA.APPEND 

This function lets you to write values and create records based on the scope. If there are existing values in the target scope, the new values are appended to them. 
### Syntax 

```
DATA.APPEND [<measures/dimension>=<member_id>,...],[<date_dim>=<date_scope_function>])

```

### Syntax Elements 

```
<member_id> ::= { <leaf_member> | <#VARIABLEMEMBER> | <parameter_single_setHier_leaf> | <parameter_single_anyHier_leaf> | [<linked_model>.]<dimension> | [<linked_model>.]<dimension>.<property> | ELIMMEMBER() }
<date_scope_function> ::= [ PERIOD () | NEXT() | PREVIOUS () | FIRST () | PREYEARLAST () ]

```

### Examples 

```
DATA.APPEND([d/Account] = "90011001")) = RESULTLOOKUP([d/Account] = "11101000") + RESULTLOOKUP([d/Account] = "22000000")

```

```
VARIABLEMEMBER #Account OF [d/Account]
DATA.APPEND([d/Account] = #Account) = RESULTLOOKUP([d/Account] = "90011001") * 1.2

```

```
DATA.APPEND([d/Measures] = "Amount") = RESULTLOOKUP([d/Measures] = "Quantity") * RESULTLOOKUP([d/Measures] = "Price")

```

```
DATA.APPEND([d/Account] = %InputAccount%) = RESULTLOOKUP([d/Account] = "90011001") * 1.2

```

```
DATA.APPEND([d/Entity] = [d/CostCenter].[p/Region]) = RESULTLOOKUP([d/Entity] = "HQ_Europe")

```

```
DATA.APPEND([ d/Entity = ELIMMEMBER([d/ENTITY], "CHINA", "KOREA", [d/ENTITY].[p/Elimination] = "Y")) = RESULTLOOKUP()

```

# Calculation Expressions 

These expressions include functions for reading data within the default model or from a different model, and for carrying forward values over multiple time periods. 
---

## RESULTLOOKUP 

This function returns a data set of corresponding, specified dimension values filtered by the formula. 
### Syntax 

```
RESULTLOOKUP ([<dimension_all>=<member_id>,...],[<date_dim>=<date_scope_function>])

```

### Syntax Elements 

```
<dimension_all> ::= <measures/dimension> | <version_dim> 
<member_id> ::= { <member> | <#VARIABLEMEMBER> | <parameter_single_setHier_leaf> | <parameter_single_anyHier_leaf> | <dimension> | <dimension_all>.<property> | ELIMMEMBER() }
<date_scope_function> ::= [ PERIOD () | NEXT() | PREVIOUS () | FIRST () | PREYEARLAST () ]

```

### Examples 

```
DATA([d/Account] = "90011001")) = RESULTLOOKUP([d/Account] = "11101000") + RESULTLOOKUP([d/Account] = "22000000")

```

```
VARIABLEMEMBER #Account OF [d/Account]
DATA([d/Account] = "90011001") = RESULTLOOKUP([d/Account] = #Account) * 1.2

```

```
DATA([d/Account] = "90011001") = RESULTLOOKUP([d/Account] = %InputAccount%) * 1.2

```

```
DATA([d/Entity] = "HQ_Europe") = RESULTLOOKUP([d/Entity] = [d/CostCenter].[p/Region])

```

```
DATA([d/Measures] = "Amount") = RESULTLOOKUP([d/Measures] = "Quantity") * RESULTLOOKUP([d/Measures] = "Price")

```

```
DATA() = RESULTLOOKUP([d/Date] = PREVIOUS())

```

```
DATA() = RESULTLOOKUP([d/Date] = NEXT(12))

```

---

## LINK 

The LINK function is used to copy or read data from a different model than the default model in the data action. You can only write data to the default model, so LINK functions can't be defined in DATA functions to write data to the linked model.
LINK functions support classic account models and models with measures (with or without an account dimension) as either the source or target model. You can copy across any of these different model types. 
After the linked model scope is defined, the LINK function can be used in a way to offer more functionality and flexibility to cross-model data actions. 
The linked mode scope, which can also be called source model scope, determines the dimension members that will be inherited by the dimension filters in the LINK function and other functions like IF and FOREACH. Note that only missing dimensions can be used in the linked model scope defined by MODELâ€¦ENDMODEL, but not all missing dimensions need to be defined in it. If you don't define a missing dimension this way, all its members will be used by the dimension filters in other functions such as LINK. In the LINK function itself, you don't need to select one member for a missing dimension in the function, which means they can be omitted in the LINK function. 
### Syntax 

```
LINK (<linked_model>,<linked_version>,<linked_member>{,...},[<shared_member>,...])

```

### Syntax Elements 

```
<linked_model> ::= Linked <MODEL> 
<linked_version> ::= {<version_dim>=<leaf_member>} 
<linked_member> ::= <linked_dimension>= {<member> | <dimension>.<property>} 
  <linked_dimension> ::= Local <measures/dimension> of linked MODEL 
<shared_member> ::= <shared_dimension>= {<member> | <dimension>.<property> | <date_scope_function>} 
  <shared_dimension> ::= Public <dimension> that exists in both the linked and default MODEL. 
  <date_scope_function> ::= [ PERIOD () | NEXT() | PREVIOUS () | FIRST () | PREYEARLAST () ]

```

### Examples 

```
DATA() = LINK([BusinessPlanning_AF], [d/Version] = "Plan01", [d/Account] = [d/ChartOfAccount].[p/SUB_Acc], [d/Entity] = "REG0001",
              [d/CostCenter] = "GENERAL", [d/Product] = "PRD0002", [d/Customer] = "CUST0001", [d/Flow] = "F_OPE")

```

```
DATA() = LINK([BusinessPlanning_Universal], [d/Version] = "Plan01", [d/Measures] = "Quantity", [d/Entity] = "REG0001",
              [d/CostCenter] = "GENERAL", [d/Product] = "PRD0002", [d/Customer] = "CUST0001", [d/Flow] = "F_OPE")

```

```
DATA([d/Account_Fin] = "Revenue_Domestic-Product ", [d/DataSrc] = "DataSrc_Sales") = LINK([CDR_Uni_Sales], [d/Measures] = "Quantity", [d/Version] = "public.Plan01") 
                                                                                   * LINK([CDR_Uni_Sales], [d/Measures] = "Price", [d/Version] = "public.Plan01")

// Product_Item and Sales_Channel are omitted in the definition of LINK function.

```

---

## ATTRIBUTE 

Returns the values of a dimension property. The values must be numbers. 
### Syntax 

```
ATTRIBUTE ([<linked_model>.]<dimension>.<property>,[<member_id>])

```

### Syntax Elements 

```
<member_id> ::= <leaf_member> | <parent_member> | <parameter_single_setHier_leaf>| <parameter_single_anyHier_leaf>

```

### Examples 

```
@Volume = ATTRIBUTE([d/Product].[p/Volume], "Prod_001")

```

```
MEMBERSET [d/Product] = ("Prod_002", "Prod_003")
DATA() = RESULTLOOKUP() * ATTRIBUTE([d/Product].[p/Volume])

```

```
IF ATTRIBUTE([d/Employee].[p/Payment_Year]) > 2000 THEN
    DATA() = RESULTLOOKUP() * 1.05
ENDIF

```

```
DATA() = ATTRIBUTE([d/Employee].[p/Payment_Month])

```

```
MODEL [Sales]
ENDMODEL

@Volume = ATTRIBUTE([Sales].[d/Product_Item].[p/ProductUnit], "ACAC")

```

---

## CARRYFORWARD 

This business function allows users to carry forward values to the opening amount of new periods. This is a very common way for business users to manage the cash flow of balance sheet statement accounts. 
The carry forward function only changes the opening values and closing values (or an alternative target member, if you define one). 
### Syntax 

```
CARRYFORWARD(<measures/dimension>,<opening_member>,<closing_member>,<closing_calc>,[<target_member>])

```

### Syntax Elements 

```
<opening_member> ::= <leaf_members/measures> FOR Opening balance 
<closing_member> ::= <leaf_members/measures> FOR Closing balance 
<closing_calc> ::= <leaf_members/measures> [ [ <operator> <leaf_members/measures> ] ...] 
<target_member> ::= <leaf_members/measures> FOR target

```

### Examples 

```
MEMBERSET [d/Date] = "202001" TO "202012"
DATA() = CARRYFORWARD([d/FLOW], "OPENING", "ENDING", "OPENING" + "CHANGE" + "OTHERS")

```

```
MEMBERSET [d/Date] = "202001" TO "202012"
DATA() = CARRYFORWARD([d/Measures], "Opening", "Closing", "Opening" + "Increase" - "Decrease")

```

# Time/Period Scope Selection Functions 

These functions return a time period or a date. 
---

## PREVIOUS 

This function returns the time period or a date that deducts a specified offset value (a signed integer) of a certain granularity from the current date dimension, a dimension attribute of date format, or a fixed date. 
### Syntax 

```
PREVIOUS([<offset>],[<time_granularity>],[<date>])

```

### Syntax Elements 

```
<offset> ::= <signed_integer> | <@INTEGER> | <parameter_number> 
<time_granularity> ::= "DAY" | "MONTH" | "QUARTER" | "YEAR" 
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
MEMBERSET [d/Date] = PREVIOUS(1, "MONTH", [d/Version].[p/StartPeriod]) TO [d/Version].[p/EndPeriod]

```

```
DATA([d/Date] = PREVIOUS(@INTEGER)) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(1, "MONTH", "2020-04-21"))

```

```
DATA([d/Date] = PREVIOUS(%ExternalParameter%, "MONTH", "2020-04-21")) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/Date] = PREVIOUS(2, "QUARTER", %DateMember%))

```

```
DATA([d/Date] = PREVIOUS(3, "MONTH", TODAY())) = RESULTLOOKUP()

```

```
MODEL [BOM]
ENDMODEL

DATA([d/Date] = PREVIOUS(1, "MONTH", [BOM].[d/Date])) = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")

```

---

## NEXT 

This function returns the time period or a date that adds a specified offset value (a signed integer) of a certain granularity to the current date. 
### Syntax 

```
NEXT([<offset>],[<time_granularity>],[<date>])

```

### Syntax Elements 

```
<offset> ::= <signed_integer> | <@INTEGER> | <parameter_number>
<time_granularity> ::= "DAY" | "MONTH" | "QUARTER" | "YEAR" 
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
MEMBERSET [d/Date] = NEXT(1, "MONTH", [d/Version].[p/StartPeriod]) TO [d/Version].[p/EndPeriod]

```

```
DATA() = RESULTLOOKUP([d/Date] = NEXT())

```

```
DATA([d/Date] = NEXT(@INTEGER)) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/Date] = NEXT(1, "MONTH", "2020-04-21"))

```

```
DATA([d/Date] = NEXT(%ExternalParameter%, "MONTH", "2020-04-21")) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/Date] = NEXT(2, "QUARTER", %DateMember%))

```

```
DATA([d/Date] = NEXT(3, "MONTH", TODAY())) = RESULTLOOKUP()

```

```
MODEL [BOM]
ENDMODEL

DATA([d/Date] = NEXT(1, "MONTH", [BOM].[d/Date])) = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")

```

---

## FIRST 

This function returns the first period of the year taken from the given date argument. 
### Syntax 

```
FIRST([<date>])

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <time_parameter_single_setHier_leaf> | [<linked_model>.]<time_dim> | [<linked_model>.]<dimension>.<date_attribute> | TODAY()
<time_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <time_dim>
<date_attribute> ::= <ATTRIBUTE> of Date type

```

### Examples 

```
DATA([d/Date] = FIRST()) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Volume", [d/Date] = FIRST())

```

```
DATA() = RESULTLOOKUP([d/Date] = FIRST([d/Employee].[p/Startdate])

```

```
DATA([d/Date] = FIRST(%TimeMember%)) = RESULTLOOKUP()

```

```
MODEL [BOM]
ENDMODEL

DATA([d/Date] = FIRST([BOM].[d/Date])) = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")

```

---

## LAST 

This function returns the last time period of the year taken from the given date argument. 
### Syntax 

```
LAST ([<date>])

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <time_parameter_single_setHier_leaf> | [<linked_model>.]<time_dim> | [<linked_model>.]<dimension>.<date_attribute> | TODAY()
<time_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <time_dim>
<date_attribute> ::= <ATTRIBUTE> of Date type

```

### Examples 

```
DATA([d/Time] = LAST()) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Volume", [d/Time] = LAST())

```

```
DATA() = RESULTLOOKUP([d/Date] = LAST([d/Employee].[p/Startdate])

```

```
DATA([d/Date] = LAST(%TimeMember%)) = RESULTLOOKUP()

```

```
MODEL [BOM]
ENDMODEL

DATA([d/Date] = LAST([BOM].[d/Date])) = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")

```

---

## PREYEARLAST 

This function returns the last period of the year prior to the year taken from the given date argument. 
### Syntax 

```
PREYEARLAST([<date>])

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <time_parameter_single_setHier_leaf> | [<linked_model>.]<time_dim> | [<linked_model>.]<dimension>.<date_attribute> | TODAY()
<time_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <time_dim>
<date_attribute> ::= <ATTRIBUTE> of Date type

```

### Examples 

```
DATA([d/Date] = PREYEARLAST()) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Volume", [d/Date] = PREYEARLAST())

```

```
DATA() = RESULTLOOKUP([d/Date] = PREYEARLAST([d/Employee].[p/Startdate])

```

```
DATA([d/Date] = PREYEARLAST(%TimeMember%)) = RESULTLOOKUP()

```

```
MODEL [BOM]
ENDMODEL

DATA([d/Date] = PREYEARLAST([BOM].[d/Date])) = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")

```

---

## TODAY 

This function returns current system's date in Coordinated Universal Time (UTC) in the format YYYYMMDD. 
### Syntax 

```
TODAY()

```

### Examples 

```
MEMBERSET [d/Date] = "202001" TO PERIOD(TODAY())

```

```
DATA([d/Date] = PERIOD(TODAY())) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/Account] = "Volume", [d/Date] = PERIOD(TODAY()))

```

```
IF [d/Date] = PERIOD(TODAY()) THEN
    DATA() = RESULTLOOKUP([d/Account] = "Volume")
ENDIF

```

# Datetime Functions 

Use datetime functions to convert dates and time periods into numbers in your advanced formulas. 
---

## DAY 

This function returns the day in a date. The day will be in number format such as 31. 
### Syntax 

```
DAY(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() * DAY([d/Employee].[p/EndDate])

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Salary") * DAY("2019-03-15")

```

```
IF DAY(%DateMember%) = 31 THEN
    DATA() = RESULTLOOKUP() + DAY([d/Date])
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA() = RESULTLOOKUP() * DAY([BOM].[d/Date])

```

---

## WEEK 

This function returns the week in a date. The week will be in number format such as 11. 
### Syntax 

```
WEEK(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() * WEEK([d/Employee].[p/EndDate])

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Salary") * WEEK("2021-03-15")

```

```
IF WEEK(%DateMember%) = 11 THEN
    DATA() = RESULTLOOKUP() + WEEK([d/Date])
ENDIF

```

```
IF WEEK([d/Employee].[p/StartDate]) = 11 THEN
    DATA() = RESULTLOOKUP() * 1.1
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA() = RESULTLOOKUP() * WEEK([BOM].[d/Date])

```

---

## MONTH 

This function returns the month in a date. The month will be in number format such as 12. 
### Syntax 

```
MONTH(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY()
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() * MONTH([d/Employee].[p/EndDate])

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Salary") * MONTH("2019-03-15")

```

```
IF MONTH(%DateMember%) = 1 THEN
    DATA() = RESULTLOOKUP() + MONTH([d/Date])
ENDIF

```

```
IF MONTH([d/Employee].[p/StartDate]) = 3 THEN
    DATA() = RESULTLOOKUP()
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA() = RESULTLOOKUP() * MONTH([BOM].[d/Date])

```

---

## YEAR 

This function returns the year in a date. The year will be in number format such as 2019. 
### Syntax 

```
YEAR(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY()
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() + YEAR([d/Employee].[p/EndDate])

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Salary") * YEAR([d/Date])

```

```
IF YEAR(%DateMember%) = 2020 THEN
    DATA() = RESULTLOOKUP() + MONTH([d/Date])
ENDIF

```

```
IF YEAR([d/Employee].[p/Startdate]) = 2021 THEN
    DATA() = RESULTLOOKUP()
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA() = RESULTLOOKUP() + YEAR([BOM].[d/Date])

```

---

## DAYSINMONTH 

This function returns the number of days in a calendar month taken from a specific date, date dimension, or dimension property. The number returned is an integer. 
### Syntax 

```
DAYSINMONTH(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY()
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() * DAYSINMONTH([d/Employee].[p/EndDate])

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Salary") * DAYSINMONTH([d/Date])

```

```
IF DAYSINMONTH(%DateMember%) = 30 THEN
    DATA() = RESULTLOOKUP() + MONTH([d/Date])
ENDIF

```

```
IF DAYSINMONTH([d/Employee].[p/ EndDate]) = 30 THEN
    DATA() = RESULTLOOKUP() / 30 * DAY([d/Employee].[p/EndDate])
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA() = RESULTLOOKUP() * DAYSINMONTH([BOM].[d/Date])

```

---

## DAYSINYEAR 

This function returns number of days in a calendar year. The year is a 4-digit number taken from a specific date, date dimension, or dimension property. 
### Syntax 

```
DAYSINYEAR(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY()
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() * DAYSINYEAR([d/Employee].[p/EndDate])

```

```
DATA() = RESULTLOOKUP([d/Account] = "Salary") * DAYSINYEAR([d/Date])

```

```
IF DAYSINYEAR(%DateMember%) = 365 THEN
    DATA() = RESULTLOOKUP() + MONTH([d/Date])
ENDIF

```

```
IF DAYSINYEAR([d/Employee].[p/EndDate]) = 366 THEN
    DATA() = RESULTLOOKUP() * DAYSINMONTH([d/Employee].[p/EndDate]) / DAYSINYEAR([d/Employee].[p/EndDate])
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA() = RESULTLOOKUP() + DAYSINYEAR([BOM].[d/Date])

```

---

## DATERATIO 

This function calculates how many days between the start and end date overlap the given period, and then divides the overlapping days by the total number of days in the period. 
### Syntax 

```
DATERATIO(<start_date>,<end_date>,[<date>])

```

### Syntax Elements 

```
<start_date> ::= <fixed_date> | [<linked_model>.]<dimension>.<date_property> | TODAY()
<end_date> ::= <fixed_date> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | <date_dim> | <dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
DATA() = RESULTLOOKUP() * DATERATIO([d/Employee].[p/ StartDate], [d/Employee].[p/EndDate], TODAY())

```

```
DATA([d/Account] = "AnnualSalary") = RESULTLOOKUP([d/Account] = "AnnualSalary") / 12 * DATERATIO([d/Employee].[p/StartDate], [d/Employee].[p/EndDate], [d/Date])

```

```
IF DATERATIO([d/Employee].[p/StartDate], [d/Employee].[p/EndDate], [d/Date]) > 0.5 THEN
    DATA() = RESULTLOOKUP()
ENDIF

```

```
MODEL [HR]
ENDMODEL

DATA() = RESULTLOOKUP() * DATERATIO([HR].[d/Employee].[p/ StartDate], [HR].[d/Employee].[p/EndDate], TODAY())

```

---

## DATEDIFF 

This function returns the period of time between two dates of a specified time granularity. The granularity can be day, month, quarter or year. 
### Syntax 

```
DATEDIFF(<date_1>,<date_2>,<time_granularity>,[<calculation_mode>])

```

### Syntax Elements 

```
<date_1> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_2> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format 
<time_granularity> ::= "DAY" | "WEEK" | "MONTH" | "QUARTER" | "YEAR" 
<calculation_mode> ::= "CalendarDiff" | "FLOOR" | "Ceiling"

```

### Examples 

```
DATA() = RESULTLOOKUP() * DATEDIFF([d/Employee].[p/ StartDate], [d/Employee].[p/EndDate], TODAY())

```

```
DATA([d/Account] = "AnnualSalary") = RESULTLOOKUP([d/Account] = "AnnualSalary") / 365 * DATEDIFF([d/Employee].[p/StartDate], [d/Employee].[p/EndDate], [d/Date])

```

```
IF DATEDIFF([d/Date], [d/Bond].[p/StartDate], "MONTH", "CalendarDiff") = 0 THEN
    DATA() = RESULTLOOKUP() * ATTRIBUTE([d/Bond].[p/Interest_Rate])
ENDIF

```

```
MODEL [BOM]
ENDMODEL

IF DATEDIFF([d/Date], [BOM].[d/ValidTo], "MONTH") >= 0 THEN
    IF DATEDIFF([d/Date], [BOM].[d/Date], "MONTH") <= 0 THEN
        DATA([d/ProductGroup_P] = "SKU") = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")
    ENDIF    
ENDIF

```

---

## PERIOD 

This function converts a date to a time period based on the Date granularity. 
### Syntax 

```
PERIOD(<date>)

```

### Syntax Elements 

```
<date> ::= <fixed_date> | <date_parameter_single_setHier_leaf> | [<linked_model>.]<date_dim> | [<linked_model>.]<dimension>.<date_property> | TODAY() 
<date_parameter_single_setHier_leaf> ::= <parameter_single_setHier_leaf> of <date_dim> 
<date_property> ::= <property> with date format

```

### Examples 

```
MEMBERSET [d/Date] = PERIOD(TODAY()) TO "202012"

```

```
DATA([d/Date] = PERIOD([d/Employee].[p/StartDate]) = RESULTLOOKUP()

```

```
DATA() = RESULTLOOKUP([d/ACCOUNT] = "Volume", [d/Date] = PERIOD(%DateMember%))

```

```
IF [d/Date] = PERIOD([d/Employee].[p/EndDate]) THEN
    DATA() = RESULTLOOKUP([d/ACCOUNT] = "Salary")
ENDIF

```

```
MODEL [BOM]
ENDMODEL

DATA([d/Date] = PERIOD([BOM].[d/Date])) = LINK([BOM], [d/Version] = "public.Plan01", [d/Measures] = "Quantity")

```

# Iteration Functions 

These functions let you repeat one or more statements iteratively over a specific scope of dimension members, or a specified number of times. 
---

## FOREACH 

This function repeats a group of statements over the FOREACH scope defined by one or more dimensions or dimension attributes. 
### Syntax 

```
FOREACH [<linked_model>.]<dimension>[.<property>][ASC|DESC][,...] 
  <Statement>
ENDFOR

```

### Examples 

```
FOREACH [d/Date]
    DATA() = RESULTLOOKUP([d/DATE] = PREVIOUS(1)) * 1.1
ENDFOR

```

```
FOREACH [d/Date].[p/YEAR]
    DATA([d/Flow]="F_OPE") = RESULTLOOKUP([d/Flow]="F_CLO"),[d/DATE] = PREVIOUS(12))
ENDFOR

```

```
MODEL [BOM]
ENDMODEL

FOREACH [BOM].[d/Product_BOM]
    //Statement...
ENDFOR

```

---

## FOREACH.BOOKED 

This function repeats a group of statements for each booked combination of dimension members in a specified scope of one or more dimensions. Compared to FOREACH, this function can improve performance by avoiding unnecessary repetitions on unbooked members. 
### Syntax 

```
FOREACH.BOOKED [<linked_model>.]<dimension>[.<property>][ASC|DESC][,...] 
  <Statement>
ENDFOR

```

### Examples 

```
FOREACH.BOOKED [d/Entity].[p/Region]
    DATA() = RESULTLOOKUP([d/DATE] = PREVIOUS(1)) * 1.1
ENDFOR

```

---

## FOR 

This function repeats a group of statements a specified number of times. 
### Syntax 

```
FOR <counter> = <start_value> TO <end_value> [STEP <step_value>]
  <Statement>
ENDFOR

```

### Syntax Elements 

```
<counter> ::= <integer_variable_name>
<start_value> ::= <signed_integer> | <integer_variable_name>
<end_value> ::= <signed_integer> | <integer_variable_name>
<step_value> ::= <signed_integer>

```

### Examples 

```
INTEGER @counter

FOR @counter = 1 TO 10 STEP 2
   DATA() = RESULTLOOKUP() * 0.5
ENDFOR

```

```
INTEGER @counter

FOR @counter = 10 TO 1 STEP -2
   DATA() = RESULTLOOKUP() * 0.5
ENDFOR

```

```
INTEGER @counter
INTEGER @endValue

@endValue = %Input_End_Number%

FOR @counter = 1 TO @endValue
   DATA() = RESULTLOOKUP() * 0.5
ENDFOR

```

---

## BREAK 

To stop the loop based on specific conditions, you can add BREAK to a FOREACH or FOREACH.BOOKED loop. When the condition is met, the remaining calculations in the loop are skipped. 
### Syntax 

```
BREAK

```

### Examples 

```
FOREACH [d/Date]
    IF RESULTLOOKUP([d/Account] = "REVENUE") <= 0
        BREAK
    ENDIF
    <Statement>
    [...]
ENDFOR

```

# Condition Statements 

These keywords let you set conditions on a group of statements so that they only run in specific cases. 
---

## IF ELSEIF ELSE 

IF statements, combined with ELSEIF and ELSE statements, let you conditionally run a group of statements depending on the value of an expression. By combining conditions using OR or AND operators, you can create complex conditions. 

### Limitations 


Measure filters with OR operators are not supported. 

When you include a measure filter and a non-measure filter, ELSEIF clauses aren't supported. 

For RESULTLOOKUP() functions in a model with measures, a measure needs to be included in the RESULTLOOKUP() function. 

### Syntax 

```
IF <condition> {[<and_or> <condition>]} THEN 
  <Statement> 
  [...] 
[ELSEIF <condition> {[<and_or> <condition>]}] 
  <Statement> 
  [...]
[ELSE] 
  <Statement> 
  [...]  
ENDIF

```

### Syntax Elements 

```
<condition> ::= <comparison1> | <comparison2> | <null_check1> | <null_check2>

<comparison1> ::= <member_comparison> | <property_comparison> | <time_comparison>
<comparison2> ::= <number> [[<operator> <number>]...] <comparator_all> <number_time> [[<operator> <number_time>]...] 
<null_check1> ::= <dimension>.<property> <comparator><""> 
<null_check2> ::= {RESULTLOOKUP() | LINK()} <comparator> <null>
	
<member_comparison> ::= {[<linked_model>.]<measures/dimension> <comparator> <member_id>} 
<member_id> ::= {(<leaf_member/measure>[,...]) | <dimension_property> | <member_parameter> | (<BASEMEMBER>[,...])} 
  <dimension_property> ::= <dimension > | <dimension >.<property>
  <member_parameter> ::= <parameter_any_setHier_leaf> | <parameter_any_anyHier_leaf> | <parameter_single_setHier_leaf> | <parameter_single_anyHier_leaf>
  <BASEMEMBER> ::= BASEMEMBER() 
<property_comparison> :: = {<dimension>.<property> <comparator> <property_value>[,...]} 
<time_comparison> ::= {<date_dim> <comparator> <date_member_id>} 
<date_member_id> ::= [<date_member> | <dimension>.<property>]

<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() | LINK() 
<number_time> ::= <numeric_value> | <numeric_time_functions>| | RESULTLOOKUP() | ATTRIBUTE() | LINK() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <mathematical_functions> | <conversion_functions> 
<numeric_time_functions> ::= <mathematical_functions> | <conversion_functions> | <time_calculation>

<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC() 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF()

```

### Examples 

```
IF RESULTLOOKUP([d/Account] = "PRICE") > 0 THEN
    //Statement... 
ENDIF

```

```
IF ABS(@NUM) > 0 OR @Total = 20 THEN
    //Statement... 
ENDIF

```

```
IF [d/Account].[p/Acctype] = ("AST", "LEQ") THEN
    //Statement... 
ENDIF

```

```
IF [d/Account] = "REVENUE" AND [d/Product] = "16GB" THEN
    //Statement... 
ELSEIF [d/Product] = "32GB" THEN
    //Statement...
ENDIF

```

```
IF [d/Flow] = (BASEMEMBER([d/Flow].[h/Hierarchy], "F_TOTAL")) THEN
    //Statement... 
ENDIF

```

```
IF [d/Measures] = ("Quantity", "Price") THEN
    //Statement... 
ENDIF

```

```
IF [d/Measures] = "Price" AND [d/CostCenter] = "Germany" THEN
    //Statement...
ENDIF

```

```
IF [d/ORG] == [d/Factory].[p/Region] THEN
    //Statement...
ENDIF

```

```
IF [d/ORG].[p/currency] !== [d/PRODUCT].[p/currency] THEN
    //Statement...
ENDIF

```

```
IF [d/Measures] = "Price" THEN
    //Statement...
ELSE
    //Statement...
ENDIF

```

```
IF RESULTLOOKUP([d/Measures] = "Price", [d/Product_Group] = "AC") > 1000 THEN
    //Statement...
ENDIF

```

```
MODEL [Sales]
ENDMODEL

IF [Sales].[d/Product_Item] = "SKU_001" THEN
    //Statement...
ENDIF

```

```
MODEL [Sales]
ENDMODEL

IF [d/Product] == [Sales].[d/Product].[p/Detail] THEN
    //Statement...
ENDIF

```

```
MODEL [Production]
ENDMODEL

IF [d/Entity].[p/Region] !== [Production].[d/Factory].[p/Region] THEN
	//Statement...
ENDIF

```

```
MODEL [Sales]
ENDMODEL

IF LINK([Sales], [d/Version] = "public.Actual", [d/Measures] = "S90007000") < LINK([Sales], [d/Version] = "public.Plan01", [d/Measures] = "S90007000") THEN
    //Statement...
ENDIF

```

```
MODEL [BOM]
ENDMODEL

IF DATEDIFF([d/Date], [BOM].[d/ValidTo], "MONTH") >= 0 THEN
    //Statement...
ENDIF

```

```
 * Unsupported Example.

IF [d/Measures] != "Quantity" OR [d/CostCenter] = "Germany" THEN
    //Statement...
ENDIF

```

```
 * Unsupported Example.

IF [d/Measures] = "Price" AND [d/CostCenter] = "Germany" THEN
    //Statement...
ELSEIF [d/CostCenter] = ("Germany", "France") THEN
    //Statement...
ENDIF

```

# Numeric Functions 

These functions let you carry out mathematical calculations and conversion functions on your data. 
---

## ABS 

This function returns the absolute value of a number. 
### Syntax 

```
ABS(<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = ABS(@FLOAT)

```

```
DATA() = ABS(RESULTLOOKUP()) * ABS(%Number%)

```

```
DATA() = RESULTLOOKUP() + ABS(ATTRIBUTE([d/INTCO].[p/Number]))

```

---

## LOG 

This function returns the natural logarithm. 
### Syntax 

```
LOG(<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = LOG(@FLOAT)

```

```
DATA() = LOG(ABS(RESULTLOOKUP())) * LOG(ABS(%Number%))

```

```
DATA() = RESULTLOOKUP() + LOG(ABS(ATTRIBUTE([d/IntCo].[p/Number])))

```

---

## LOG10 

This function returns the base 10 logarithm. 
### Syntax 

```
LOG10(<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT()
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = LOG10(ABS(@FLOAT))

```

```
DATA() = LOG10(ABS(RESULTLOOKUP())) * LOG10(ABS(%Number%))

```

```
DATA() = RESULTLOOKUP() + LOG10(ABS(ATTRIBUTE([d/INTCO].[p/Number])))

```

---

## MOD 

This function returns the remainder after the first number is divided by the second number. The result has the same sign the second number. 
### Syntax 

```
MOD(<number>,<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = MOD(@FLOAT, @int)

```

```
DATA() = MOD(RESULTLOOKUP(), %Number%)

```

```
DATA() = MOD(@FLOAT, RESULTLOOKUP())

```

```
DATA() = RESULTLOOKUP() + MOD(ATTRIBUTE([d/IntCo].[p/Number]), 10)

```

---

## POWER 

This function returns the first number raised to the power of the second number. 
### Syntax 

```
POWER(<number>,<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = POWER(@FLOAT, @int)

```

```
DATA() = POWER(RESULTLOOKUP(), %Number%)

```

```
DATA() = POWER(@FLOAT, RESULTLOOKUP())

```

```
DATA() = RESULTLOOKUP() + POWER(ATTRIBUTE([d/IntCo].[p/Number]), 10)

```

---

## SQRT 

This function returns the square root of a number. 
### Syntax 

```
SQRT(<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = SQRT(ABS(@FLOAT))

```

```
DATA() = SQRT(ABS(RESULTLOOKUP())) * SQRT(ABS(%Number%))

```

```
DATA() = RESULTLOOKUP() + SQRT(ABS(ATTRIBUTE([d/INTCO].[p/Number])))

```

---

## ROUND 

This function rounds a number to a specified number of decimal places. 
### Syntax 

```
ROUND(<number>,<integer>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()
<INTEGER> ::= <signed_integer> | <@INTEGER> | <parameter_number>

```

### Examples 

```
DATA() = ROUND(@FLOAT, @int)

```

```
DATA() = ROUND(%Number%, - 2)

```

```
DATA() = ROUND(RESULTLOOKUP(), %Number%)

```

```
DATA() = RESULTLOOKUP() + ROUND(ATTRIBUTE([d/IntCo].[p/Number]), 2)

```

---

## FLOOR 

This function lets you specify a number and an integer. It returns a number that has the number of decimal places specified by the integer, and that is as large as possible without being greater than the entered number. The integer is optional and defaults to 0 if you don't specify it. 
### Syntax 

```
FLOOR(<number>,<integer>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC() 
<INTEGER> ::= <signed_integer> | <@INTEGER> | <parameter_number>

```

### Examples 

```
DATA() = FLOOR(RESULTLOOKUP())

```

```
DATA() = FLOOR(@FLOAT, @int)

```

```
DATA() = FLOOR(%Number%, - 2)

```

```
DATA() = FLOOR(RESULTLOOKUP(), %Number%)

```

```
DATA() = RESULTLOOKUP() + FLOOR(ATTRIBUTE([d/INTCO].[p/Number]), 2)

```

---

## CEIL 

This function lets you specify a number and an integer. It returns a number that has the number of decimal places specified by the integer, and that is as small as possible without being smaller than the entered number. The integer is optional and defaults to 0 if you don't specify it. 
### Syntax 

```
CEIL(<number>,<integer>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC() 
<INTEGER> ::= <signed_integer> | <@INTEGER> | <parameter_number>

```

### Examples 

```
DATA() = CEIL(RESULTLOOKUP())

```

```
DATA() = CEIL(@FLOAT, @int)

```

```
DATA() = CEIL(%Number%, - 2)

```

```
DATA() = CEIL(RESULTLOOKUP(), %Number%)

```

```
DATA() = RESULTLOOKUP() + CEIL(ATTRIBUTE([d/IntCo].[p/Number]), 2)

```

---

## TRUNC 

This function returns a numeric valueâ€¯truncated to a specific numberâ€¯of decimal places, represented by the integer. 
### Syntax 

```
TRUNC(<number>,<integer>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE() 
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number> 
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions> 
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF() 
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT() 
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC() 
<INTEGER> ::= <signed_integer> | <@INTEGER> | <parameter_number>

```

### Examples 

```
DATA() = TRUNC(@FLOAT, @int)

```

```
DATA() = TRUNC(%Number%, - 2)

```

```
DATA() = TRUNC(RESULTLOOKUP(), %Number%)

```

```
DATA() = RESULTLOOKUP() + TRUNC(ATTRIBUTE([d/IntCo].[p/Number]), 2)

```

---

## fx-FLOAT 

This function returns the floating-point number of a given number. (Note that the FLOAT keyword can also be used to define a variable with a floating-point data type.) 
### Syntax 

```
FLOAT(<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE()
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number>
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions>
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF()
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT()
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = FLOAT(@FLOAT)

```

```
DATA() = FLOAT(RESULTLOOKUP()) * FLOAT(%Number%)

```

```
DATA() = RESULTLOOKUP() + FLOAT( ATTRIBUTE([d/INTCO].[p/Number]))

```

---

## fx-INT 

This function returns the integer portion of a given number. 
### Syntax 

```
INT(<number>)

```

### Syntax Elements 

```
<number> ::= <numeric_value> | <numeric_functions> | RESULTLOOKUP() | ATTRIBUTE()
<numeric_value> ::= <numeric> | <number_variable> | <parameter_number>
<numeric_functions> ::= <time_calculation> | <mathematical_functions> | <conversion_functions>
<time_calculation> ::= DAY() | MONTH() | YEAR() | DAYSINMONTH() | DAYSINYEAR() | DATERATIO() | DATEDIFF()
<mathematical_function> ::= ABS() | LOG() | LOG10() | MOD() | POWER() | SQRT()
<conversion_function> ::= ROUND() | FLOOR() | CEIL() | TRUNC()

```

### Examples 

```
DATA() = INT(@FLOAT)

```

```
DATA() = INT(RESULTLOOKUP()) * INT(%Number%)

```

```
DATA() = RESULTLOOKUP() + INT( ATTRIBUTE([d/INTCO].[p/Number]))

```
Impressum Disclaimer Copyright Cookie Preferences 
Copyright 
Disclaimer 
Privacy Statement 
Legal Disclosure 
Trademark 
Terms of Use