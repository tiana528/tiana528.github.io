---
title: hive GenericUDF and DeferredJavaObject analysis
date: 2019-01-09 22:06:25
tags:
---

# Background
This article aims at discussing how hive generic User-defined function(GenericUDF) works. In the java doc, it says GenericUDF can do short-circuit evaluations using DeferedObject. But what is short-circuit evaluation and how DeferedObject works?

# Description
Basicly, use GenericUDF when the input or output is complex type, or input arguments have variable length.

GenericUDF use DeferedObject to pass arguments and achieve lazy-evaluation and short-circuiting.
``` java
  public static interface DeferredObject {
    void prepare(int version) throws HiveException;
    Object get() throws HiveException;
  };
```


# Source code analysis
We run a sql `select abs(c1) from t1` and try to analysis how UDF is initialized and processed.

- Class structure and relationship
    - SelectOperator
        - ExprNodeEvaluator[] eval
            - (actually it is ExprNodeGenericFuncEvaluator)
    - ExprNodeGenericFuncEvaluator
        - ExprNodeEvaluator[] children; 
            - (actually it is [ExprNodeColumnEvaluator[Column[c1]]])
        - GenericUDF.DeferredObject[] deferredChildren;
            - (actually it is org.apache.hadoop.hive.ql.exec.ExprNodeGenericFuncEvaluator$DeferredExprObject and contains the ExprNodeEvaluator)
        - GenericUDF.DeferredObject[] childrenNeedingPrepare;
            - (actually it contains the same object as deferredChildren in this example)
        - GenericUDF genericUDF;
            - (actually it is GenericUDFAbs)




- During initialization

    - Generate query plan tree
        - SemanticAnalyzer.genPlan
    - Create ExprNodeGenericFuncDesc based on genericUDFClass
```java
org.apache.hadoop.hive.ql.plan.ExprNodeGenericFuncDesc
 public static ExprNodeGenericFuncDesc newInstance(GenericUDF genericUDF,
      String funcText,
      List<ExprNodeDesc> children) {
    ...
    ObjectInspector oi = genericUDF.initializeAndFoldConstants(childrenOIs);
    ...
}
```
    - Initialize GenericUDF
```java
org.apache.hadoop.hive.ql.udf.generic.GenericUDF
public ObjectInspector initializeAndFoldConstants(ObjectInspector[] arguments)
      throws UDFArgumentException {

    ObjectInspector oi = initialize(arguments);
    ...
}
```
    ```java
org.apache.hadoop.hive.ql.udf.generic.GenericUDFAbs
  public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
    ...
}
```
    - Initial select operator
        - Create ExprNodeGenericFuncEvaluator
```java
org.apache.hadoop.hive.ql.exec.SelectOperator
@Override
  protected void initializeOp(Configuration hconf) throws HiveException {
  ...
  eval[i] = ExprNodeEvaluatorFactory.get(colList.get(i), hconf);
  ...
}

```
        ```java
org.apache.hadoop.hive.ql.exec.ExprNodeGenericFuncEvaluator
  public ExprNodeGenericFuncEvaluator(ExprNodeGenericFuncDesc expr, Configuration conf) throws HiveException {
    children = new ExprNodeEvaluator[expr.getChildren().size()];
    for (int i = 0; i < children.length; i++) {
      ExprNodeDesc child = expr.getChildren().get(i);
      ExprNodeEvaluator nodeEvaluator = ExprNodeEvaluatorFactory.get(child, conf);
      children[i] = nodeEvaluator;
      ...
    }
}

```
        - Init ExprNodeGenericFuncEvaluator and create DeferredExprObject
```java
org.apache.hadoop.hive.ql.exec.ExprNodeGenericFuncEvaluator
  @Override
  public ObjectInspector initialize(ObjectInspector rowInspector) throws HiveException {
     deferredChildren = new GenericUDF.DeferredObject[children.length];
     
}
```
        ```java
ExprNodeGenericFuncEvaluator.DeferredExprObject
 DeferredExprObject(ExprNodeEvaluator eval, boolean eager) {
      this.eval = eval; //(ExprNodeEvaluator[Column[c1]])
      this.eager = eager; (false)
    }
    ```



- During processing data
    - select operator process one row
```java
org.apache.hadoop.hive.ql.exec.SelectOperator
  @Override
  public void process(Object row, int tag) throws HiveException {
    ...
    output[i] = eval[i].evaluate(row); //(eval[i] is actually ExprNodeGenericFuncEvaluator)
    ...
}
```
        - Note that `row` is an LazyStruct object which holds data in binary format

    ```java
org.apache.hadoop.hive.ql.exec.ExprNodeGenericFuncEvaluator
@Override
  protected Object _evaluate(Object row, int version) throws HiveException {
    ...
    return genericUDF.evaluate(deferredChildren);
  }
```
    ```java
org.apache.hadoop.hive.ql.udf.generic.GenericUDFAbs
  @Override
  public Object evaluate(DeferredObject[] arguments) throws HiveException {
    Object valObject = arguments[0].get();
    ...
}```
        - Note that arguments[0] is ExprNodeGenericFuncEvaluator&DeferredExprObject

    ```java
org.apache.hadoop.hive.ql.exec.ExprNodeGenericFuncEvaluator
 @Override
    public Object get() throws HiveException {
     ...
     obj = eval.evaluate(rowObject, version); //(eval is ExprNodeColumnEvaluator)
     ...
   }
```
    ```java
org.apache.hadoop.hive.ql.exec.ExprNodeColumnEvaluator
 @Override
  protected Object _evaluate(Object row, int version) throws HiveException {
   ...
   return inspector.getStructFieldData(row, field); //(inspector is LazySimpleStructObjectInspector)
    
}
```


# Summarize
- Processing
    - During intilization
        - hive creates SelectOperator, SelectOperator contains ExprNodeGenericFuncEvaluator, ExprNodeGenericFuncEvaluator contains GenericUDFAbs and DeferredExprObject (DeferredObject), DeferredExprObject contains ExprNodeColumnEvaluator (column[c1]).
    - During executing
        - SelectOperator processes row which is LazyStruct, and at last passed to LazySimpleStructObjectInspector (getStructFieldData) to get the actual data from binary data.
- lazy-evaluation and short-circuiting
    - We can notice that the value of the attribute which is involved in the UDF calculation is only evaluated just before use. The value is analyzed from binary format. This is very efficient.


# Cache approach in GenericUDF
```java
 @Override
  public Object evaluate(DeferredObject[] arguments) throws HiveException {
    Object obj = arguments[0].get();
    ...
}
```
The obj got DeferredObject in GenericUDF evaluate function is  an LazyInteger object, it is always the same object even if evaluate function is invoked for processing a different row.

We should be careful when making use of cache to speedup GenericUDF calculation. Buffer the input and output, and returns buffered output if historical input comes. Since we always get the same object from the DeferredObject, if we use `equals` function to compare, it will always be true. We need to deep copy the inputs for further comparison. 

