####Encoders
  
   Encoders是Spark中对DataSet的record进行序列化/反序列化(SerDe)框架，它将Java对象/原生类型(PrimitiveType)序列化成SparkSQL中的InternalRow，或者从InternalRow中反序列化为Java对象/原始类型。
   
   Spark代码中trait Encoder[T]就是实现上述功能的一个接口，如下：
   
  ```
  	trait Encoder[T] extends Serializable {
  		def schema: StructType
		def clsTag: ClassTag[T]
	}
  ```  
   其中`T`表示DataSet中record的实际类型(Java对象/原生类型)，通过`Encoder[T]`可以进行序列化/反序列化的操作，它相当于是`DataSet[T]`的record的序列化/反序列化功能的容器。由于`Encoder[T]`包含schema信息，所以它的序列化/反序列化在性能上比JavaSerializer和KryoSerializer要好。
   
   Spark会根据数据类型`T`来默认生成对应的`Encoder[T]`(通过SparkSession.implicits._)，如下：
   
   ```
   import spark.implicits._
   val ds = Seq(1, 2, 3).toDS()
   ```
   `Seq(1, 2, 3)`中元素类型为`Int`，Spark会自动调用`SQLImplicits`中的`newIntEncoder`得到`Int`这种原生类型的`Encoder[Int]`，后续`ds`中元素的序列化/反序列的操作就由`Encoder[Int]`来负责。
   
   Spark2.0支持的`Encoder[T]`的类型如下：
   
| 支持的Encoder类型 |
| ---- |
| Product子类 |
| Int/Long/Double/Float/Byte/Short/Boolean/String |
| Seq/Array |

   
  `ExpressionEncoder`是`trait Encoder[T]`的唯一实现类：
  
  ```
  case class ExpressionEncoder[T](
    schema: StructType,
    flat: Boolean,
    serializer: Seq[Expression],
    deserializer: Expression,
    clsTag: ClassTag[T])
  extends Encoder[T]{
    //代码略
  }
  ```
  其中serializer/deserializer就是用来实现对`T`的序列化(`T` -> `InternalRow`)/反序列化(`InternalRow` -> `T`)
   
  SparkSql的执行计划的执行实际是一堆算子利用`Encoder[T]`来`读(反序列化)->处理->写(序列化))`DataSet中的record的过程。
  
    
 ![](https://github.com/windpiger/sparkstudy/blob/master/pic/internalrowlogic.png)
   
   
####InternalRow

   InternalRow是SparkSQL内部进行计算逻辑时使用的数据结构，通过Encoder对java对象/原生数据序列化/反序列化成InternalRow，它的类图关系如下：
   
   ![](https://github.com/windpiger/sparkstudy/blob/master/pic/internalrow.png)
   
   
   Encoder序列化/反序列化逻辑:
   
   ![](https://github.com/windpiger/sparkstudy/blob/master/pic/expressionencoder.png)

   
   * UnSafeRow
   
   	 Spark将java对象/原生类型的数据序列化成UnSafeRow，以二进制的形式存储record，**UnsafeRow的存储形式相对于java serializer的占用空间更小，且结合schema信息可以直接从二进制中获取对应类型的数据，存取速度更快。**
   	 
   	 ![](https://github.com/windpiger/sparkstudy/blob/master/pic/unsaferow.png)
   
   * SpecificInternalRow
     
     MutableValue类型存储容器，MutableValue类型的数据可以重复使用，减少GC。
     
    `class SpecificInternalRow(val values: Array[MutableValue])`
   * GenericInternalRow
     
      `class GenericInternalRow(val values: Array[Any])`
      
#### CodeGen

   Encoder中的serializer和deserializer其实是Expression类型，最终执行的时候会通过CodeGen动态将Expression转换成javacode。
   
   Expression是如何转换成javacode？
   
   Expression类中有个genCode函数 如下:
   
   ```
     def genCode(ctx: CodegenContext): ExprCode = {
    ctx.subExprEliminationExprs.get(this).map { subExprState =>
      // This expression is repeated which means that the code to evaluate it has already been added
      // as a function before. In that case, we just re-use it.
      ExprCode(ctx.registerComment(this.toString), subExprState.isNull, subExprState.value)
    }.getOrElse {
      val isNull = ctx.freshName("isNull")
      val value = ctx.freshName("value")
      val ve = doGenCode(ctx, ExprCode("", isNull, value))
      if (ve.code.nonEmpty) {
        // Add `this` in the comment.
        ve.copy(code = s"${ctx.registerComment(this.toString)}\n" + ve.code.trim)
      } else {
        ve
      }
    }
  }
   
   ```
   
   Expression本质上就是对input进行表达式计算然后输出结果，所以可以将Expression的输出结果用一个变量value_n来表示，而实际的计算过程是根据具体的Expression实现类来具体实现的，即Expression的子类需要实现上述代码中的`doGenCode`方法，以`Add`这个Expression为例：
   
   ```
     override def symbol: String = "+"

     override def doGenCode(ctx: CodegenContext, ev: ExprCode): ExprCode = dataType match {
    case dt: DecimalType =>
      defineCodeGen(ctx, ev, (eval1, eval2) => s"$eval1.$$plus($eval2)")
    case ByteType | ShortType =>
      defineCodeGen(ctx, ev,
        (eval1, eval2) => s"(${ctx.javaType(dataType)})($eval1 $symbol $eval2)")
    case CalendarIntervalType =>
      defineCodeGen(ctx, ev, (eval1, eval2) => s"$eval1.add($eval2)")
    case _ =>
      defineCodeGen(ctx, ev, (eval1, eval2) => s"$eval1 $symbol $eval2")
  }
   ```
  `defineCodeGen` 逻辑省略，从上述代码可以看出大致的逻辑，即：
  
  - DecimalType类型的java代码为使用plus函数
  - ByteType/ShortType的java代码为直接使用+号
 
 对于复杂的Expression，比如有多个嵌套的child的表达式，是通过深度优先遍历（后序遍历递归）的方式逐步处理，最终得到所需的java代码，如下所示：
 
  ![](https://github.com/windpiger/sparkstudy/blob/master/pic/codegen.png)
 
   即每个子child会生成javacode，并且会用一个变量value_i来表示该子child在javacode处理后的返回值，改返回值即可以提供该父Expression生成javacode时来使用。
