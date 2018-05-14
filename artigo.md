# Como utilizamos URL schemes no PlayKids iOS

## Introdução
 
A URL scheme é uma funcionalidade muito interessante pois permite que aplicativos comuniquem entre em si. É possível clicar em um link no Safari e ser redirecionado para um app, levar o usuário diretamente para um conteúdo, etc.  

O próprio iOS possui URL schemes nativas, sendo possível iniciar o app de telefone discando para um número, iniciar o app de e-mail com o destinatário escrito, etc.

No PlayKids a funcionalidade é utilizada em diversas ocasiões:

### Engajamento & Deep Link
Na nossa página da web, área dos pais e redes sociais temos diversos links que levam para um conteúdo específico do app. É possível abrir um vídeo, playlist ou jogos de qualquer personagem diretamente.

Também enviamos push notifications com dicas de conteúdos. O push possui uma URL scheme dentro do payload, que quando aberto redireciona diretamente ao conteúdo.

### Suporte
Temos URLs para alterar configurações do app e enviar diagnóstico de problemas. Essas URLs são utilizadas pela nossa equipe de suporte para auxiliar clientes com problemas. É muito mais simples direcionar o usuário diretamente para onde ele precisa ir do que enviar um passo-a-passo por e-mail.

### Comunicação com TV App
Fomos convidados a integrar nosso aplicativo com o TV App da Apple para o lançamento no Brasil. O TV App é um agregador de conteúdo e funciona praticamente como um controle remoto, onde o usuário escolhe o episódio/show que quer assistir e é redirecionado para o app que possui o conteúdo.


Apresentamos aqui uma forma simples mas eficaz para você introduzir as URL schemes em seu app!

## Utilizando URL schemes

Antes de qualquer coisa, é preciso cadastrar qual scheme seu app vai utilizar no **Info.plist**. Em nosso exemplo vamos utilizar **demoapp://** mas é recomendado que você escolha algo que faça sentido com o nome do seu app.

Clique com o botão direito em `Information Property List`, selecione `Add Row` e escolha `URL types`. Clique com o botão direito em `Item 0`, selecione `Add Row` e escolha `URL Schemes`. Verifique se ficou assim:

![Info.plist](https://i.imgur.com/UtjBVyj.png)

Em **URL identifier** é geralmente utilizado o mesmo valor do bundle identifier.

Em **URL Schemes** é definido os schemes. Defina apenas um que seja único pois em caso de conflito com outro app no sistema, é possível que o iOS não escolha o seu.

Para facilitar a implementação, vamos utilizar um framework que ajuda na organização e leitura das URL schemes, o **[JLRoutes](https://github.com/joeldev/JLRoutes)**. A nossa ideia é definir uma rota para cada URL scheme que vamos suportar e para isso precisamos criar um protocolo e uma estrutura:

```swift
import JLRoutes

protocol URLSchemeRoute {
    var routePatterns: [String] { get }
    func routeWithParameters(_ parameters: RouteParameters) -> Bool
}

struct RouteParameters {
    let route: String
    private let parameters: [AnyHashable: Any]

    subscript(_ key: String) -> Any? {
        return parameters[key]
    }

    init?(parameters: [AnyHashable: Any]) {
        var copyParameters = parameters
        if let route = copyParameters.removeValue(forKey: JLRoutePatternKey) as? String {
            self.route = route
            self.parameters = copyParameters
        } else {
            return nil
        }
    }
}
```

**URLSchemeRoute** representa a nossa rota. Ela define quais URLs são suportadas e possui um método para que seja feito o roteamento quando uma URL é recebida.

**RouteParameters** são os parâmetros extraídos da URL recebida pelo sistema. Esses parâmetros podem vir através de query strings ou como parte do path da URL.

Como exemplo, vamos criar uma rota chamada **PlayVideoNavigationRoute**, que toca um vídeo passado como parâmetro no path da URL:

```swift
struct PlayVideoNavigationRoute: URLSchemeRoute {
    // Definimos a URL scheme e indicamos que a mesma necessita de um parâmetro videoId
    // Por exemplo: demoapp://play_video/abcd1234
    // É possível definir mais de um padrão, assim como parâmetros opcionais
    // Leia a documentação do JLRoutes para aprender todas as possibilidades
    let routePatterns = ["play_video/:videoId"]

    func routeWithParameters(_ parameters: RouteParameters) -> Bool {
        guard let videoId = parameters["videoId"] as? String else { return false }

        print("tocar vídeo com id \(videoId)")
        return true
    }
}
```

Agora que temos a nossa rota, precisamos registra-la no JLRoutes. Isso pode ser feito diretamente no AppDelegate, mas para deixar o código mais organizado, preferimos criar uma classe para fazer essa ponte:

```swift
import JLRoutes

final class URLSchemeManager {
    func registerRoute(_ route: URLSchemeRoute) {
        JLRoutes.global().addRoutes(route.routePatterns) { parameters -> Bool in
            return RouteParameters(parameters: parameters).map(route.routeWithParameters) ?? false
        }
    }

    func openURL(_ url: URL) -> Bool {
        return JLRoutes.routeURL(url)
    }
}

struct URLSchemeManagerFactory {
    static func createURLSchemeManager() -> URLSchemeManager {
        let urlSchemeManager = URLSchemeManager()
        urlSchemeManager.registerRoute(PlayVideoNavigationRoute())
        return urlSchemeManager
    }
}
```

Em **URLSchemeManagerFactory** criamos nosso objeto e registramos as rotas. Por curiosidade, no PlayKids esse factory é feito em um arquivo separado pois temos muitas rotas registradas. Consegue adivinhar o número?  Para finalizar, ficou assim o AppDelegate:

```swift
import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    let urlSchemeManager = URLSchemeManagerFactory.createURLSchemeManager()

    func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey: Any] = [:]) -> Bool {
        return urlSchemeManager.openURL(url)
    }
}
```

Registramos as rotas assim que o app é iniciado e quando uma URL scheme é recebida, passamos ela para o JLRoutes que por sua vez a transmite para alguma de nossas rotas com os parâmetros já extraídos.

Pronto! Para testar, basta digitar a URL completa no Safari e o mesmo vai redirecionar para o seu app.

## Um aplicativo feito apenas com URL schemes

Ao adotar URL schemes você abre portas para novas formas de engajamento e benefícios para seus usuários mas elas também podem ser um desafio para desenvolver.   

Tecnicamente é possível que elas sejam executadas a qualquer momento e nem sempre o app está preparado para isso. Questionamentos do tipo “Funcionaria executar uma URL para tocar um vídeo enquanto estiver na tela desse jogo?” são comuns e precisam ser testados pelo desenvolvedor.  

Outro ponto para prestar atenção é a falta de testes. Se a funcionalidade não é utilizada com freqüência e não existem testes automatizados, é possível que se quebre com o tempo e o problema só é descoberto em produção.  

Para mitigar esses problemas pensamos em uma abordagem diferente. E se toda a navegação do app fosse feita através de URLs? Se ao invés de criar view controllers e passar parâmetros entre eles, por que não criar rotas e simplesmente chamar a URL?   

O PlayKids seguiu por esse caminho e é por isso que temos 77 rotas registradas atualmente, sendo possível acessar qualquer conteúdo ou tela a partir de outro app. Como tudo é feito através das URLs, temos como requisito resolver o problema da imprevisibilidade desde cedo e o resultado até o momento (2 anos de uso) tem se mostrado bem positivo.  

Em um próximo post vamos mostrar como organizamos a navegação do nosso app para tornar possível o uso máximo dessa funcionalidade. Fiquem ligados =)


## Referências

**Inter-App Communication**
https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW2
