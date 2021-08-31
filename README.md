# the-java-code-manipulation
코드를 조작하는 다양한 방법 

이번 강의에서 다룬 내용 
* JVM 구조 
* 바이트 코드 조작 -ASM , Javassist, ByteBuddy(내부적으로 ASM 사용)
* 다이나믹 프록시 기법 - Proxy 라는 클래스를 사용할수도 있지만, 경우에 따라서 CGlib, ByteBuddy / 하이버네이트가 어떻게 동작하는지 라이브러리 프레임워크가 이해하는데 도움이 된다.
* 리플렉션 API -정말 많이 사용된다. 한번 로딩된 클래스에 대한 정보를 참조할때 클래스에 대한 직접적인 레퍼런스 없이 클래스 인스턴스를 만들다던가 / 클래스 정보 참조(메소드, 필드, 생성자, …) <제네릭> 도 참조 가능 , 다른 기술하고도 같이 사용할 수도 있다. 어떤 클래스에 어떤 필드가 있느냐 private 필드도 접근이 가능하다. 무분별하게 사용하면 성능적 이슈가 있을 수 있다. 
* 애노테이션 프로세서 - AbstractProcessor, Filer, processEnv , RoundEnvironment, TypeElement 도 학습해야 한다. , AutoService,JavaPoet(자바 소스 코드를 코딩으로 프로그래밍으로 소스코드를 생성하려고 할때 유용)


복습 정리노트




애노테이션으로 모자안의 토끼를 꺼내는 방법 

클래스들을 컴파일 할때 처리할 수 있는 프로세서를 만들어야 한다.   
AbstractProcessor 를 상속받아서 
Process 를 재정의해준다 . 

여기서 오버라이딩 해야 하는 몇가지 메서드들이 있기 때문에 이걸 사용하면 된다.   
<pre>
<code>
@Retention(RetentionPolicy.SOURCE)
public @interface Magic {
    // 애노티에션
}
</code>
</pre>

<pre>
<code>
@AutoService(Processor.class)
public class MagicMojaProcessor extends AbstractProcessor {  //AbstractProcessor을 상속받는다. 

@Override
public Set<String> getSupportedAnnotationTypes()  {
    return Set.of(Magic.class.getName());
}

@Override
public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.latestSupported();
}

@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Magic.class);
        for (Element element : elements) {
            Name elementName = element.getSimpleName();  
            if (element.getKind() != ElementKind.INTERFACE) {    // Element Kind 를 쓰면 어떤 형태로 쓰이는지 알수 있다. 
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, "Magic Annotation can not be used on " + elementName);  // 에러처리를 해야한다 그럴때 쓸수 있는게 processingEnv 에 들어있는 getMessager 를 쓸수 있다.
            } else {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, "Processing " + elementName);
            }

            // Annotation 을 검사했고 믿에는 소스코드를 생성하는 라인   
            TypeElement typeElement = (TypeElement)element; //엘리먼트를 타입엘리먼트로 변환
            ClassName className = ClassName.get(typeElement); //타입 엘리먼트를 클래스 네임으로 변환 (javapoet)

            MethodSpec pullOut = MethodSpec.methodBuilder("pullOut")    //메서드 구현
                    .addModifiers(Modifier.PUBLIC)
                    .returns(String.class)
                    .addStatement("return $S", "Rabbit!")
                    .build();

            TypeSpec magicMoja = TypeSpec.classBuilder("MagicMoja")     //클래스 구현
                    .addModifiers(Modifier.PUBLIC)
                    .addSuperinterface(className)                       //상속 받는 부분
                    .addMethod(pullOut)
                    .build();
            //소스파일을 쓰는 과정 
            Filer filer = processingEnv.getFiler(); //AbstractProcessor 를 상속받은 클래스들은 processingEnv 를 쓸수있다. Filer 를 쓸수있다. 핵심코드 

            try {
                JavaFile.builder(className.packageName(), magicMoja)    // JavaFile을 만들때 Builder magicMoja 라는 타입을 저 패키지 안에 만들겠다.
                        .build()                                        //만들어달라
                        .writeTo(filer);                                // filer 에 쓰게한다.
            } catch (IOException e) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, "FATAL ERROR: " + e);       // 에러처리
            }
        }
        return true; // true 를 리턴하면 이 애노테이션 타입을 처리 한 것이다. 그래서 다음 프로세서들 한테 더이상 이 애노테이션을 처리하라고 부탁하지 않는다. 경우에 따라서 다른곳에서도 처리할것 같으면 false 를 처리한다. 
}
}
</code>
</pre>

jar 패키징을 하려면 resource 를 등록하는 등 상당히 번거로운데 그것을 도와주는 자동으로 컴파일하주는 AutoServices 를 사용하면 된다.    
클래스 위에 @AutoService(Processor.class) 를 적으면 
메뉴페스트 파일을 애노테이션 프로세서를 통해서 이클래스를 컴파일 할때 META-INF.services 안에 파일을 자동으로 만들어준다.   
빌드하면 이제 jar 파일이 만들어져 있는데 이것을 어노테이션 적용할 프로젝트에 dependecy 시키면 된다. 

ServiceProvider 라는 개념    
* 확장포인트를 제공하는 개념 이다. ? 

JavaPoet : 소스 코드 생성 유틸리티 이다.   
굉장히 직관적이다. 메서드를 만들고 클래스를 만들수 있다. 

Filer 인터페이스   
* 소스코드, 클래스 코드 및 리소스를 생성할 수 있는 인터페이스   (핵심적클래스)

이렇게 하면 다른 프로젝트에서 
MagicMoja.class 가 생기지만 사용하려면 IntelliJ 나 IDE 에서 Annotation Processors 을 사용옵션을 켜줘야 한다.   
그리고 Project Settings 에서 Modules -> tartget/generated-sources/annotations 를 Sources 로 적용시켜준다.   

