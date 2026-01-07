# Powershell에 대해 알아봅시다

안녕하십니까. 이번에 PowerShell을 배우면서 Powershell 튜토리얼이 인트라넷이 있으면 좋을 것 같다는 생각이 들어서 이 글을 작성하게 되었습니다.

PowerShell은 마이크로소프트가 개발한 쉘 스크립트로, 무려 Windows 운영체제에 기본 탑재가 됩니다. 즉, 인트라넷 PC에서도 사용할 수 있다는 장점이 있습니다.

PowerShell은 Windows Powershell과 그냥 Powershell로 나뉘는데 기본으로 설치되어 있는 PowerShell은 Windows PowerShell입니다. Windows PowerShell은 마이크로소프트에서 더이상 업데이트 하지 않으며, 최신 버전은 5.1입니다.

## Powershell의 특징

Powershell은 텍스트를 입력받고 반환하는 대부분의 shell(예: bash)과 달리 .NET object를 입력받고 반환합니다.

.NET은 마이크로소프트가 개발한 무료 오픈 소스 플랫폼으로, 런타임(CLR, Java의 JVM같은 거라고 보면 됩니다), 표준 라이브러리(BCL), 기타 등등이 포함됩니다. .NET에 속하는 언어는 대표적으로 C#, F#, VB 등이 있습니다.

.NET object는 다음과 같은 특징을 가집니다.

+ .NET 타입 시스템에 속함
+ CLR(Common Language Runtime)에서 관리됨
+ `System.Object`를 최상위 기반으로 함.

즉, 모든 .NET object는 개념적으로 다음과 같은 계층에 속합니다.

```
System.Object
 ├─ System.String
 ├─ System.Int32
 ├─ System.IO.FileInfo
 ├─ System.Diagnostics.Process
 └─ 사용자 정의 클래스 (C#, F# 등)
```

## Hello, world!

프로그래밍 언어는 hello world로 시작하는게 국룰이니, 저도 hello world로 시작해 보겠습니다. 일단 아래의 아이콘처럼 생긴 프로그램(Windows PowerShell ISE)을 Windows 검색에서 찾아 들어가줍니다. Windows에 기본적으로 설치되어 있는 PowerShell 전용 IDE라고 보시면 됩니다.

![image-20260105184619497](asset/powershell_icon.png)

들어가면 파란색 프롬프트가 뜰텐데, 그곳에 아래 명령어를 입력해줍니다. 이 명령어를 실행해야 PowerShell 스크립트 파일을 실행시킬 수 있습니다.

```powershell
Set-ExecutionPolicy RemoteSigned
```

> [!NOTE]
>
> 이 cmdlet(명령)은 Windows의 PowerShell 실행 정책을 바꾸며, 이는 PowerShell 보안 전략의 일부입니다. 위 명령은 해당 PC에서 PowerShell 스크립트 실행을 허용하게 될겁니다. 하지만, 사실 실행 정책은 PowerShell "스크립트" 실행을 막습니다. PowerShell 실행은 막지 않으므로, 보안에 주는 영향은 아주 **미미**합니다.

만약에 관리자 권한이 없어서 오류가 뜬다면 아래 명령어를 대신 실행해주시기 바랍니다.

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

파일을 만들고 아래 코드를 입력해 봅시다.

```powershell
Write-Host "Hello, world!"
```

그다음 키보드의 `F5` 버튼 또는 상단의 초록색 삼각형 화살표를 눌러줍시다. 그럼 하단의 CMD 창에 `Hello, world!`가 출력되게 됩니다. PowerShell의 세계에 발을 들이신 것을 축하합니다!

> [!NOTE]
>
> PowerShell 파일의 확장자는 `ps1`인데, 이 파일은 보통은 더블 클릭으로 실행되지 않으며, 파일을 우클릭하여 `PowerShell에서 실행` 버튼을 선택해서 실행해야 합니다.

이제 코드를 분석해 보시죠.

### cmdlet

hello world 코드에서 `Write-Host`는 cmdlet(커맨드릿이라고 읽음)이란 것입니다. cmdlet은 독립적인 실행 프로그램이 아니라, PowerShell의 기본 명령입니다. cmdlet은 필요에 따라 로드할 수 있는 PowerShell 모듈로 구성됩니다. cmdlet은 대부분 C#으로 작성된 .NET 클래스이며 이 클래스들은 `Cmdlet` 또는 `PSCmdlet`을 상속받습니다. C#으로 cmdlet 동적 라이브러리를 만들고, PowerShell이 그것을 로드하여 사용한다고 보시면 됩니다.

PowerShell은 `Verb-Noun`(동사-명사) 쌍을 사용하여 cmdlet의 이름을 지정합니다. `Write-Host`도 그렇게 구성된 것을 볼 수 있습니다. PoWeRsHeLl은 일반적으로(명확한 기준은 잘 모르겠습니다) 대소문자를 구분하지 않으며, 따라서 `WrItE-HoSt`라고 써도 아무 문제가 없습니다. 그렇다고 `"Hello, world!"`를 `"HeLlO, WoRlD"`라고 쓰면 그대로 출력됩니다.

`Write-Host`는 .NET object를 받아서 문자열화(내부적으로 `ToString()` 호출)하여 콘솔에 출력합니다.

### 인자

`"Hello, world!"`는 `System.String`이라는 타입을 가지는 .NET 객체의 리터럴입니다. PowerShell은 `System.String` 객체를 생성하고 이를 `Write-Host` cmdlet의 `-Object` 매개변수에 바인딩합니다. cmdlet의 매개변수의 타입은 지정되어 있으며, PowerShell은 바인딩중에 (타입이 일치하지 않을시) 타입변환을 시도합니다.

#### Parameter Binding

PowerShell의 *parameter binding*은 여러 종류가 있으며, 다음 순서에 의해 일어납니다.

1. named parameter binding
2. positional parameter binding
3. pipeline parameter binding
   1. ByValue
   2. ByPropertyName
4. default value

##### Named Parameter Binding

가장 명시적인 방식입니다.

```powershell
Get-Process -Name powershell -Id 1234
```

특징으론 순서에 의존하지 않으며, 부분 이름을 허용합니다(예: `-Na` -> `-Name`). 하지만 모호하면 오류가 발생합니다.

##### Positional Parameter Binding

parameter에 position 속성이 정의된경우 사용됩니다.

```powershell
Get-Process powershell
```

위는 내부적으로 아래와 같습니다.

```powershell
Get-Process -Name powershell
```

##### Pipeline Parameter Binding

이에 대해서는 파이프라인 편에서 알아보겠습니다.

##### Default Value

말그대로 기본값입니다.

### cmdlet 구현 예시

```C#
[Cmdlet(VerbsCommunications.Write, "Host")] // cmdlet의 이름을 지정
public class WriteHostCommand : PSCmdlet // PSCmdlet을 상속받음
{
    // 속성(property)가 파라미터라는 것을 지정
    // Position = 0은 positional parameter binding에서 첫 번째 위치임을 뜻함
    [Parameter(Position = 0)]
    public object Object { get; set; } // 속성 이름은 파라미터 이름이기도 함

    protected override void ProcessRecord() // cmdlet이 실행되면 호출됨
    {
        if (Object == null)
		    return;
		print(Object.ToString()); // 출력
    }
}
```

## 끝내며

이번 PowerShell 튜토리얼은 여기서 마치겠습니다. 다음엔 "help 시스템을 사용해보자" 편으로 만나뵙겠습니다.
