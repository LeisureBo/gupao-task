## 代理模式作业

### 为什么JDK动态代理中要求目标类实现的接口数量不能超过65535个？

    这个是Java 虚拟机规范规定的。
    首先要知道Class类文件结构：
       * Class文件是一组以8字节为基础单位的二进制流，
       * 各个数据项目严格按照顺序紧凑排列在class文件中，
       * 中间没有任何分隔符，这使得class文件中存储的内容几乎是全部程序运行的程序。

    ClassFile 结构体，如下：
       ClassFile {
           u4             magic;
           u2             minor_version;
           u2             major_version;
           u2             constant_pool_count;
           cp_info        constant_pool[constant_pool_count-1];
           u2             access_flags;
           u2             this_class;
           u2             super_class;
           u2             interfaces_count;
           u2             interfaces[interfaces_count];
           u2             fields_count;
           field_info     fields[fields_count];
           u2             methods_count;
           method_info    methods[methods_count];
           u2             attributes_count;
           attribute_info attributes[attributes_count];
       }

     各项的含义描述：
     1，无符号数，以u1、u2、u4、u8分别代表1个字节、2个字节、4个字节、8个字节的无符号数
     2，表是由多个无符号数或者其它表作为数据项构成的复合数据类型，所以表都以“_info”结尾，由多个无符号数或其它表构成的复合数据类型

    看上边第18行 interfaces_count 接口计数器，interfaces_count 的值表示当前类或接口的直接父接口数量。类型是u2， 2个字节，即 2^16-1 = 65536-1 = 65535
    所以目标类实现的接口数量不能超过65535个
### 仿JDK动态代理实现原理，自己手写一遍。

* DiyProxy 核心类

```
public class DiyProxy {

	public static final String ln = "\r\n";

	public static final String tab = "\t";

	// java规定动态生成的类文件以$开头
	public static final String proxyName = "$Proxy1";

	protected DiyInvocationHandler h;

	protected DiyProxy(DiyInvocationHandler h) {
		Objects.requireNonNull(h);
		this.h = h;
	}

	public static Object newProxyInstance(DiyClassLoader loader,
                Class<?>[] interfaces, DiyInvocationHandler h) throws IllegalArgumentException {
		try {

			//1、动态生成源代码.java 文件
			String src = generateSrc(interfaces);

			//2、Java 文件输出磁盘
			String filePath = DiyProxy.class.getResource("").getPath();
			System.out.println("proxy filepath: " + filePath);
			File file = new File(filePath + proxyName + ".java");
			FileWriter fw = new FileWriter(file);
			fw.write(src);
			fw.flush();
			fw.close();

			//3、把生成的.java 文件编译成.class 文件
			JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
			StandardJavaFileManager manage = compiler.getStandardFileManager(null,null,null);
			Iterable iterable = manage.getJavaFileObjects(file);
			JavaCompiler.CompilationTask task = compiler.getTask(null, manage,null,null,null, iterable);
			task.call();
			manage.close();

			//4、编译生成的.class 文件加载到 JVM 中来 (不通过 loadClass 方法加载)
			Class proxyClass = loader.findClass(DiyProxy.class.getPackage().getName() + "." + proxyName);
			Constructor c = proxyClass.getConstructor(DiyInvocationHandler.class);
			// 删除源文件
//			file.delete();

			//5、返回字节码重组以后的新的代理对象
			return c.newInstance(h);

		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	/**
	 * 模拟生成java代码（支持多个接口）
	 *
	 * @param interfaces
	 * @return
	 */
	private static String generateSrc(Class<?>[] interfaces){
		StringBuilder pg = new StringBuilder();
		pg.append("package com.bo.gupao.proxy.dynamicproxy.diyproxy;").append(ln);
		pg.append(ln);
		pg.append("import ").append("java.lang.reflect.*;").append(ln);

		StringBuilder sb = new StringBuilder();
		sb.append(ln);
		sb.append("public final class $Proxy1 extends DiyProxy implements ");
		Iterator<Class<?>> interIter = Arrays.asList(interfaces).iterator();
		while (interIter.hasNext()){
			Class<?> interface_ = interIter.next();
			// 导包
			pg.append("import ").append(interface_.getName()).append(";").append(ln);
			sb.append(interface_.getSimpleName());
			if(interIter.hasNext()){
				sb.append(", ");
			}
		}
		sb.append(ln);
		sb.append("{").append(ln);
		sb.append(tab).append("public $Proxy1(DiyInvocationHandler invocationhandler)").append(ln);
		sb.append(tab).append("{").append(ln);
		sb.append(tab).append(tab).append("super(invocationhandler);").append(ln);
		sb.append(tab).append("}").append(ln);
		sb.append(ln);

		int methodCount = 0;
		String methodNamePrefix = "m";
		Map<String, MethodDefine> methodDefineMap = new HashMap<>();
		// 遍历接口，重写接口方法
		for(Class interface_ : interfaces){
			for(Method method : interface_.getMethods()){
				MethodDefine methodDefine = new MethodDefine();
				methodDefine.setInterfaceTypeName(interface_.getName());
				methodDefine.setMethodFullName(method.getName());
				methodDefine.setReturnTypeName(method.getReturnType().getName());
				// 方法定义缓存
				String methodName = methodNamePrefix + methodCount++;
				methodDefineMap.put(methodName, methodDefine);
				// 方法定义
				sb.append(tab).append("public final ").append(method.getReturnType().getSimpleName()).append(" ").append(method.getName());
				sb.append("(");
				// 方法参数定义
				Iterator<Class<?>> paramTypes = Arrays.asList(method.getParameterTypes()).iterator();
				// 参数名称列表
				List<String> argNameList = new ArrayList<>();
				while (paramTypes.hasNext()){
					Class<?> paramType = paramTypes.next();
					// 导入方法参数包
					if(!paramType.getName().startsWith("java.lang")){
						pg.append("import ").append(paramType.getName()).append(";").append(ln);
					}
					// 方法定义增加参数类型名称
					methodDefine.addParamTypeName(paramType.getName());
					sb.append(paramType.getSimpleName()).append(" ");
					String argName = "arg" + argNameList.size();
					sb.append(argName);
					argNameList.add(argName);
					if(paramTypes.hasNext()){
						sb.append(", ");
					}
				}
				sb.append(")").append(ln);
				sb.append(tab).append("{").append(ln);
				sb.append(tab).append(tab).append("Object obj = null;").append(ln);
				sb.append(tab).append(tab).append("try").append(ln);
				sb.append(tab).append(tab).append("{").append(ln);
				sb.append(tab).append(tab).append(tab).append("obj = super.h.invoke(this, ").append(methodName).append(", ");
				// 如果参数列表部位空则添加参数数组，否则参数为null
				if(argNameList.isEmpty()){
					sb.append("null");
				}else {
					sb.append("new Object[] {");
					Iterator<String> args = argNameList.iterator();
					while (args.hasNext()){
						sb.append(args.next());
						if(args.hasNext()){
							sb.append(", ");
						}
					}
					sb.append("}");
				}
				sb.append(");").append(ln);
				sb.append(tab).append(tab).append("}").append(ln);
				sb.append(tab).append(tab).append("catch(Error _ex) { }").append(ln);
				sb.append(tab).append(tab).append("catch(Throwable throwable)").append(ln);
				sb.append(tab).append(tab).append("{").append(ln);
				sb.append(tab).append(tab).append(tab).append("throw new UndeclaredThrowableException(throwable);").append(ln);
				sb.append(tab).append(tab).append("}").append(ln);
				if(methodDefine.getReturnTypeName().equals("void")){
					sb.append(tab).append(tab).append("return;").append(ln);
				}else{
					sb.append(tab).append(tab).append("return (").append(method.getReturnType().getSimpleName()).append(")").append(" obj;").append(ln);
				}
				sb.append(tab).append("}").append(ln).append(ln);
			}
		}

		// 方法引用对象定义
		methodDefineMap.forEach((m, define) -> {
			sb.append(tab).append("private static Method ").append(m).append(";").append(ln).append(ln);
		});
		// 方法初始化静态代码块定义
		sb.append(tab).append("static").append(ln);
		sb.append(tab).append("{").append(ln);
		sb.append(tab).append(tab).append("try").append(ln);
		sb.append(tab).append(tab).append("{").append(ln);
		methodDefineMap.forEach((m, define) -> {
			sb.append(tab).append(tab).append(tab).append(m).append(" = ")
					.append("Class.forName(\"").append(define.getInterfaceTypeName()).append("\")")
					.append(".getMethod(\"").append(define.getMethodFullName()).append("\"")
					.append(", new Class[] {");
			Iterator<String> paramIter = define.getParamTypeNames().iterator();
			while(paramIter.hasNext()){
				sb.append(getClassTypeDef(paramIter.next()));
				if(paramIter.hasNext()){
					sb.append(", ");
				}
			}
			sb.append("});").append(ln);
		});
		sb.append(tab).append(tab).append("}").append(ln);
		sb.append(tab).append(tab).append("catch(NoSuchMethodException nosuchmethodexception)").append(ln);
		sb.append(tab).append(tab).append("{").append(ln);
		sb.append(tab).append(tab).append(tab).append("throw new NoSuchMethodError(nosuchmethodexception.getMessage());").append(ln);
		sb.append(tab).append(tab).append("}").append(ln);
		sb.append(tab).append(tab).append("catch(ClassNotFoundException classnotfoundexception)").append(ln);
		sb.append(tab).append(tab).append("{").append(ln);
		sb.append(tab).append(tab).append(tab).append("throw new NoClassDefFoundError(classnotfoundexception.getMessage());").append(ln);
		sb.append(tab).append(tab).append("}").append(ln);
		sb.append(tab).append("}").append(ln);

		sb.append("}").append(ln);
		return pg.append(sb).toString();
	}

	public static class MethodDefine {

		private String interfaceTypeName;

		private String methodFullName;

		private String returnTypeName;

		private List<String> paramTypeNames = new ArrayList<>();

		public String getInterfaceTypeName() {
			return interfaceTypeName;
		}

		public void setInterfaceTypeName(String interfaceTypeName) {
			this.interfaceTypeName = interfaceTypeName;
		}

		public String getMethodFullName() {
			return methodFullName;
		}

		public void setMethodFullName(String methodFullName) {
			this.methodFullName = methodFullName;
		}

		public String getReturnTypeName() {
			return returnTypeName;
		}

		public void setReturnTypeName(String returnTypeName) {
			this.returnTypeName = returnTypeName;
		}

		public List<String> getParamTypeNames() {
			return paramTypeNames;
		}

		public void setParamTypeNames(List<String> paramTypeNames) {
			this.paramTypeNames = paramTypeNames;
		}

		public void addParamTypeName(String paramTypeName){
			this.paramTypeNames.add(paramTypeName);
		}
	}

	/**
	 * 获取指定类型的class对象定义
	 *
	 * 例："int" -> "Integer.TYPE"
	 * java.lang.Object ->
	 *
	 * @param className
	 * @return
	 */
	private static String getClassTypeDef(String className){
		String classType = null;
		switch (className){
			case "byte":
				classType = "Character.TYPE";
				break;
			case "short":
				classType = "Short.TYPE";
				break;
			case "int":
				classType = "Integer.TYPE";
				break;
			case "long":
				classType = "Long.TYPE";
				break;
			case "float":
				classType = "Float.TYPE";
				break;
			case "double":
				classType = "Double.TYPE";
				break;
			case "boolean":
				classType = "Boolean.TYPE";
				break;
			case "char":
				classType = "Character.TYPE";
				break;
			case "void":
				classType = "Void.TYPE";
				break;
			default:
				classType = "Class.forName(\"" + className + "\")";
				break;
		}
		return classType;
	}
}
```

