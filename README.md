# Notes about the book Pro JPA

## Capitulo 2 - Getting Started

Para transformar um POJO em um objeto persistivel, use a anotação @Entity

### @Id

Necessario para informar ao provider qual campo é a chave primaria da entidade
Pode ser usada ou no campo ou get metodo

----------


### Persistence Context


O conjunto de todos as classes gerenciadas pelo EntityManager e o EntityManager em si, é chamado de "PesistenceContext"
Dentro de um mesmo contexto não pode existir mais de um objeto com o mesmo id

EntityManager é criado por um EntityManagerFactory que é criado atraves da classe Persistence que tem seus valores configurados na persistence unit

Quando uma entida de é retornada do banco de dados, ela passa a ser gereciada pelo EntityManager e a fazer parte do seu respectivo Persistence Context

----------

### Removendo uma entidade

Para remover uma entidade, ela precisa ja esta sendo gerenciada

**Jeito certo**

````Employee emp = em.find(Employee.class, 158);
em.remove(emp); //Caso emp seja null (não foi encontrado), será lançada uma IllegalArgumentException````

**Jeito errado**

````em.remove(new Employee(158)) //Esse objeto não faz parte do contexto de persistencia do entity manager````

Metodo "EntityManger.find" não precisa ser executado dentro de uma transação;

----


### Persistence.xml

Elemento <class> só é necessário para aplicações SE. No ambiente EE, o container escaneia por classes anotadas com @Entity

----

## Capitulo 3 - Enterprise Appplication


**Talves seja bom você pular esse capitulo agora. Muita coisa aqui não vai fazer sentido no começo**


Não é necessario  interfaces para definir interfaces de um session bean statefull, stateless ou singleton
Bens sem a anotação de tipo são locais 

### Stateless 

Tem os seguintes eventos de ciclo de vida

* **PostConstructor:**  Chamado após o termino do construtor. Ideal para a alocação de recursos
* **PreDestroy:** Chamado antes do container liberar a instancia do bean para o GC. Ideal para liberar recursos.

### Statefull

Tem os mesmos eventos de ciclo de vida do stateless bean e mais:

* **PrePassivate: ** Usado quando o server irá passivar (serializar o bean) por algum motivo (liberação de recusos, replicação para cluster, etc)
* **PostActivate:** Usado quando o container, após ativar um bean, irá ativa-lo novamente.

####@Remove

Quando um metodo anotado com @Remove é chamado, isso sinaliza para o container o fim do uso desse session statefull bean pelo cliente. Caso o cliente chame mais um algum metodo no bean após um desse metodo ter sido chamado, é lançado uma exception


### Singleton

* Podem ser criados de forma eager
* Apenas uma instancia durante toda a execucação da aplicação
* Ideal para compartilhar estados da aplicação.
* Tem os mesmos eventos de ciclo de vida de um Stateless session bean
* Definidos usando a anotação @Singleton
* O container gerencia concorrencia assegurando que apenas uma invocação por vez aconteça aos metodos do bean (ruim para performance)
* A anotação @Startup define que esse bean deve ser criado juntamente com o start da aplicação.
* Comportamentos eager ou lazy são vendor-specific, não devendo ser assumidos como verdadeiros

----

### Dependency Management and CDI

#### @Resource, @EJB, @PersistenceContext ou @PersistenceUnit


Usado nas referencias para procurar o recurso (Similar ao @Inject do CDI){}.

Essa anotação pode ser colocada no campo, setter ou, como tipicamente é, no nivel da classe.
Caso seja utilizada no nivel da classe, é obrigatorio a definição do nome (atributo "name")
Em outros lugares, um nome é gerado automaticamente pelo container.


Depedency Lookup

* Anotações de recurso a nivel de classe

A aplicação é resposavel por procurar o recurso utilizando JDNI

Exemplo:

	Context ctx = new InitialContext();
	audit = (AuditService) 
	ctx.lookup("java:comp/env/deptAudit");

O prefixo "java:comp/env" indica para o servidor onde procurar

Para EJB é possivel fazer o lookup através de EJBContext ou seu filho, SessionContext. Tem duas vantagens:

* É possivel usar diretamente o nome do recurso (context.lookup("deptAudit");)
* Esse metodo encapsula todas as exceptions checadas, ficando apenas as de runtime.


#### Dependency Injection

O container é responsavel por gerenciar as dependecias removendo assim a necessidade do lookup manualmente pela aplicação.
Para injeção através do campo, não é necessário o metodo set e o campo pode ser private
O nome gerado automatico é formado por "NOME COMPLETO DA CLASS/CAMPO"


### Declaring Dependencies


#### PersistenceContext

Modo SE de conseguir um EntityManager:

```EntityManager em = Persistence.createEntityManagerFactory(PERSISTENCE UNIT).createEntityManager();```

Modo EE de conseguir um EntityManager:

```@PersistenceContext(unitName = PERSISTENCE UNIT)
private EntityManager em;```

Nome do Persistence Unit não é obrigatorio, mas varia de vendor para vendor como tratar sua omissão (alguns, caso exista apenas um persistence unit no arquivo, usam esse. Outros obrigam o uso de um arquivo a parte)


#### PersistenceUnit

Utilizado para recuperar a instancia do EntityManagerFactory


#### Resource

Utilizado para recuperar outros recursos do servidor (datasource, initialContext/SessionContext)


##### Transaçoes

O ato de inicar (transaction.begin) e de terminar uma transaction (transaction.commit) é chamado de "transaction demarction". É um aspecto muito importante pois fazer isso maior do que o necessario, pode causar problemas de performace.


Tipos de comportamentos para um metodo com a transação gerenciada pelo container (CMT):

* MANDATORY: é esperado que quando esse metodo for chamado, já haja uma transação ativa. Caso não, uma exception é lançada;
* REQUIRED: Caso ja haja uma transação ativa para esse metodo, o container irá usa-la. Caso não, uma nova será criada.
* REQUIRED_NEW: Usado para quando o metodo precisa de uma transação para si proprio. Perigoso pois pode causar um overhead na aplicação;
* SUPORTS: não é mandatorio, mas caso um metodo seja chamado dentro de uma transação, o mesmo irá continuar rodando.
* NOT_SUPORTS: Metodo nao pode ser usado dentro de uma transação. Caso ao ser chamado uma transação ja esteja aberta, o container ira pausa-la e executar o metodo sem transação;
* NEVER: Metodo não suporta transações e faz com que o container lance uma exception caso o mesmo seja chamado dentro uma.


Transações em EJB - EJB Container-managed Transactions

EJB aceitam transações atraves de demarcação de classes e/ou metodos:


	@TransactionAttribute(TransactionAttributeType.SUPPORTS)
    public void metododeNegocio(){}

Anotação @TransactionAttribute pode ser usada na classe e/ou diretamento no metodo. Tendo no metodo, esse tem precendencia sobre classe.


#### Demais recursos - Transactional Interceptors

Para os mais recuros (beans, servlets, etc) use o inteceptor @Transcational. Segue as mesmas regras que o mecanismo usado para EJBs.

Transações não comitadas antes do fim do metodo, serão rollback pelo container. Muitas transações porem não precisam de um commit explicito pelo cliente. Quando a transação é gerenciada pelo container, a mesma será comitada no fim do seu escopo (mais sobre isso lá na frente. Aguente firme ai)


### Capitulo 4 - Object-Relational Mapping


#### Metodos de acesso

Define o modo como a propriedade será lida e mapeada pelo container e pode ser de tudas formas: diretamente pelo campo (field) ou atráves dos metodos get/set (propertie)

Não existe um padrão. Ao colocar a anotaçõe no campo ou na propriedade, você esta definindo como o campo será acessado.

Para especificar o tipo de acesso, basta colocar a anotação do JPA (@Id, @Column, etc) no local desejado.

Podem ser via campo ou via metodo:

* Campo

Anotando o campo, o mesmo será acessado via reflection. O campo não pode ser publico.

* Propriedade

Anotando o metodo, os metodo serão chamados diretamente. A anotação deve ser usada no metodo GET do campo.

O campo deve ser private ou protected.

O tipo do campo, que será mapeado para a tabela, será assumido como o tipo retornado pelo metodo. Esse tipo deve ser o mesmo do metodo set.

Usando o padrão JavaBean, é obrigatorio que o metodo GET e SET tenham o mesmo sufixo;

Ao usar esse tipo, sufixo do metodo get, é o que será usado para criar a coluna. Exemplo:

	...
	private long wage;
	public long getSalario() {return this.wage;}
	...

Nome do campo coluna será salario (getSalario) e não wage (nome do campo)
	


É possive fazer uma mistura, por exemplo, definir que o acesso padrão para todos os campos da classe é através do campo, MAS para um item espeficico que precisa de um tratamento, mudar seu acesso para metodo:


1) Use a anotação @Access e a enum AccessType para definir o acesso padrão a classe:
	
    	@Entity
		@Access(AccessType.FIELD)
		public class Employee { ... }

2) Defina que para uma propriedade especifica, o acesso deve ser feito atráves do metodo:

		@Access(AccessType.PROPERTY) 
		@Column(name="PHONE") //Importante usar essa anotação para definir qual é o nome do campo e manter o mapeamento com a tabela.
		protected String getPhoneNumberForDb() { ... }

3) Como o acesso padrão da classe é via campo, é necessario informar para o provider, que esse campo não deve ser persistido. Caso não faça isso, o valor ser persistido duas vezes (uma pela classe e outra pelo metodo):

		@Transient 
		private String phoneNum;


### Mapeando a entidade

Caso o nome da tabela não seja o mesmo da classe (o padrão ao anota a classe com @Entity), use a anotação a tabela:
	
	Esse entidade será mapeada para a tabela EMP
	@Table(name="EMP")

	É possivel usar essa anotação para definir a qual schema essa tabela pertence:
	@Table(name="EMP", schema="HR")

	O mesmo para catalogos:
	@Table(name="EMP", catalog="HR")

Existe uma diferença entre definir o nome da tabela usando a anotação Entity ou Table.

* Ao usar **Table**, o nome da tabela no banco de dados será o mesmo definido aqui.

* Ao usar **Entity**, alem do nome da tabela mudar, o nome da entidade tambem ira. O nome da entidade é usado quando estamos escrevendo queries, por exemplo.

### Mapeamento de campos

**@Basic**

A forma mais basica de falar que um campo é persistivel. 

É adicionado implicitamento.

Atráves do atributo 'fetch' pode definir que certo campo NÃO será inicialmente carregado do banco de dados. 

O mesmo só deve ser carregado quando o campo por acesso. As opções são "fetch=FetchType.LAZY"  ou "fetch=FetchType.EAGER" (padrão)

Através do atributo "nullable" é indicado se esse atributo pode ser null ou não na tabela (equivalente a "not null" na criação)

Atráves do atributo "updateable" é informado se esse atributo será atualizado no banco de dados após a execucação do EntityManager.update


**@Column**

Pode ser usada em conjunto com @Basic. Entre outras opções, pode ser usada para definir que o nome desse campo ao ser mapeado para uma coluna, deve ser outro.


### Large Objects

Para fazer o mapeamento de campos Large Objects, deve ser utilizado a anotação @Lob (Large Objects).

No banco de dados, LOB se divide em BLOB (binario) e CLOB (caracteres), sendo mapeados em java para: 

	BLOB: byte[], Byte[], e Serializable
	CLOB: char[], Character[], and String


### ENUM
Os valores declarados dentro de uma enum recebem um "index" referente a posição em que são declarados.
Ao persistir uma enum, o comportamento padrão é persitir um inteiro referente ao valor selecionado.

Exemplo:

	public enum EmployeeType {
	    FULL_TIME_EMPLOYEE, // Se usado, será persistindo o valor 0 no campo
	    PART_TIME_EMPLOYEE, // Valor 1
	    CONTRACT_EMPLOYEE;  // Valor 2
	}

Esse comportamento pode causar problemas caso a ordem da declaração da enumeção mude. Para alterar o comportamento padrão e definir que o valor da enumeração deve ser persistido como um Strig use:

		@Enumerated(EnumType.STRING)
		private EmployeeType type;


### Temporal Types
	
Quando o campo da entidade é de um tipo temporal, é necessario um tratamento adicional.

Tipos permitidos:

		java.sql.Date
		java.sql.Time
		java.sql.Timestamp
		java.util.Date
		java.util.Calendar 

Para persitir tipos do pacote java.util, é necessario usar a anotação @Temporal para especificar qual tipo do pacote java.sql o campo deve ser mapeado. Os possiveis valores são DATE, TIME, e TIMESTAMP e representa cada um dos tipo;
Ao persistir tipos do pacote javax.sql não é necessario nenhum tratamento adcional.


### Estado transiente:

Em situaçoes um campo não deve ser persistido para a base de dados, ou anote o mesmo com @Transient ou use o modificador transient.

A diferenciação de uso se deve a:

	O campo pode ser serializdo normalmente pelo java? Se sim, use a anotação. Se não, use o modificador;


### Chave primaria

Os tipos permitidos para campos definidos como chave primaria são:

	Primitive Java types: byte, int, short, long, char
	Wrapper classes of primitive Java types: Byte, Integer, Short, Long, Character
	String: java.lang.String
	Large numeric type: java.math.BigInteger
	Temporal types: java.util.Date, java.sql.Date

Ainda são permitidos os tipos double e float e seus wrappers, porem são desencorajados.


###Geração automatica de valores:

Para que o banco de dados gere automaticamente o valor da chave primaria, use a anotação @GeneretedValue e defina sua estrategia conforme melhor se adequar:

	@GeneratedValue(strategy = GenerationType.SEQUENCE||AUTO||IDENTIFY||TABLE)
	private int id;

* Opção AUTO:

Nessa opção, que a default, o ORM provider irá escolher qual é a melhor opção (das outras 4) para a geração do idenficador.
	Mais recomendada para tempo de desenvolvimento.

	* Opção TABLE:
	Ao utilizar essa opção, o valor da chave gerada para a entidade é salvo em uma tabela na base de dados.
	A tabela precisa ter duas colunas:

		* Nome do indice: String chave primaria usada para identicar o "contador"
		* Contador: Inteiro que irá guardar a ultima posição utilizada como chave primaria.

	Configuração minima para usar essa estrategia:
	
		@Id 
		@GeneratedValue(strategy=GenerationType.TABLE)
		private long id;

		Ao declarar o ID dessa forma, o nome da tabela, coluna de chave primaria e valor ficaram por conta do provider JPA e, caso a opção de schema generation estiver ativada, o mesmo ira criar esses valores. Caso não, é necessario que os valores padrões usados pelo provider sejam conhecidos e ja estejam criados.

		Resultado:

	
		Nome da tabela
		-------------------------------------------
		|COLUNA DA CHAVE PRIMARIA| COLUNA DE VALOR|
		-------------------------------------------
		| VALOR PADRAO			 |	0			  |	
		-------------------------------------------
		|						 |				  |
		-------------------------------------------

	Utilizado a anotação @TableGenerator:

	A anotação @TableGenerator pode ser definida em ATRIBUTOS ou CLASSES e serve para nomear globalmente um esquema de geração de tabelas. Caso o mesmo esquema seja usado em mais de uma entidade, é aconselhado que o mesmo seja definido via XML.
	Caso mais de um @TableGenerator aponte para mesma tabela, mas use nome de coluna para chave e valor diferentes, SOMENTE um funcionará.
	Acontece que ao subir a aplicação, o provider irá encontrar um @TableGenerator na entidade X, irá criar uma tabela com o nome especificado, e suas respectivas para chave e valor. Quando o outro @TableGenerator for encontrado na entidade Y, apontando para a mesma tabela, a tabela já existira e não serão criados as colunas para chave e valor com o nome criado. Ao persistir a entidade a entidade X, sem problemas, mas ao persistir a entidade Y, será lançado uma mensangem "a coluna X para chave (ou valor) não existe."

	As opções de utilização desse anotação são:

	1) Somente definido a anotação.
	É equivalente a configuração minina, pois todos os valores serão usados, serão os padrões definidos pelo provider JPA.
	
	@GeneratedValue(strategy = GenerationType.TABLE,generator="EMPL_GEN")
	@TableGenerator(name = "EMPL_GEN")
	@Id
	private long id;


	2) Definindo o nome da tabela e das colunas:
	Estamos dizendo que "o id do tipo tabela, será utilizara o gerador EMPL_GEN que define o nome da tabela como ID_GEN, o nome da coluna da chave primaria como GEN_NAME e nome da coluna de valor como GEN_VALUE"

	Para esse indice, o valor guardado na coluna da chave primaria será "EMPL_GEN", que é nome do gerador.

	@GeneratedValue(strategy = GenerationType.TABLE, generator="EMPL_GEN")
	@TableGenerator(
			name = "EMPL_GEN",
			pkColumnName="GEN_NAME",
			valueColumnName="GEN_VALUE",
			table="ID_GEN"
	)
	@Id
	private long id;

	Tabela ID_GEN
	-------------------------
	|GEN_NAME	| GEN_VALUE |
	-------------------------
	| <ENTIDADE>  |	0		|	
	-------------------------
-

	3) Mais opções
	Alem das opções definidas no item (2), dizemos que:
	 "o valor guardado na coluna da chave primaria (nomear do indice), não será EMPL_GEN e sim EMPLOYEE_GEN (opção pkColumnName). O valor inicial da coluna de valor, valor esse usado APENAS para criar a tabela, não será 0 e sim 1000 (ou seja, o primeiro empregado persistido terá o id 1000"

	 Para agilizar o processo, o JPA salve alguns identificadores na memoria. O tamanho inicial padrão é 50. Para alterar esse valor, use a opção 'allocationSize'.

	@TableGenerator(
		name="EMPL_GEN",
	    table="ID_GEN",
	    pkColumnName="GEN_NAME",
	    valueColumnName="GEN_VAL",
	    pkColumnValue="EMPLOYEE_GEN",
	    initialValue=10000,
	    allocationSize=100
    )
	@GeneratedValue(generator="Address_Gen")
	@Id 
	private long id;

	Tabela ID_GEN
	-----------------------------
	| GEN_NAME		| GEN_VAL   |
	-----------------------------
	| EMPLOYEE_GEN  |	10000	|	
	-----------------------------


	* Opção SEQUENCE:

	Caso o banco de dados suporte, cria uma sequencia que será utilizada para a criação da chave primaria.
	Diferente da tabela, contem apenas uma coluna (NEXT_VAL) que representa o valor a ser utilizado.
	Caso mais de uma entidade compartilhe a mesma sequencia, irão utilizar do mesmo valor incremental.
	Permitido definir apenas o valor inicial, nome da sequence (caso não fornecido, o provider usa seu padrão) e a quantidade alocada a cada vez. 
	Caso seja usado um esquema já existente, os valores dos parametros definidos acima, devem ser compativeis com os utilizados ao criar a sequence.

	Exemplo de utilização:

	1) Modo mais simples:
	Cria uma sequence com os valores padrões definidos pelo provider

	@GeneratedValue(strategy = GenerationType.SEQUENCE)
	private int id;

	2) Definindo valores
	Irá criar uma sequence (tabela na verdade) chamada "role_sequence" com parametros definidos abaixo. 
	O comando para criar a mesma sequence em SQL seria:

	CREATE SEQUENCE role_sequence MINVALUE 1 START WITH 1 INCREMENT BY 10 

	@@Id	
	@GeneratedValue(strategy = GenerationType.SEQUENCE, generator="ROLE_GEN")
	@SequenceGenerator(name="ROLE_GEN", sequenceName="role_sequence", initialValue=1, allocationSize= 10)
	private int id;


	* Opção IDENTIFY:

	É o modo mais simples de se gerar a primary key. Usa o mecanismo de AUTO INCREMENT do banco de dados, caso ele suporte.
	Após a inserção da entidade no banco de dados, o provider rele o valor e atualiza o objeto em memoria.
	Tem como desvantagem que, diferente do que acontece com os outros mecanismos que alocam os valores previamente em memoria, como a geração do valor só acontece após sua inserção, esse valor não tem como ser armazeado em memoria.
	Não é possivel compartilhar os valores entre multiplas tabelas/entidades, como acontece nas outras estrategias.

	1) Definição da estretegia:
	
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private int id;


Mapping the Primary Key
	Semelhante ao criar a tabela da seguinte forma:
	  
	create table role (
		id integer not null auto_increment,
		...
	);


* Relacionamentos - Basicos

	
	* ROLE
	
	"Há 3 lados de uma historia: o meu, o seu e a verddade"
	No relacionameto de entidades, os "lados" do relacionamento, são chamados de ROLE.
	Nesse relacionamento, empregado e projeto são as roles:
	[EMPREGADO] <----> [PROJETO]


	* DIRECIONALIDADE

	Especifica em qual direção a relação entre as roles se comunicam.

	Unidirecional:
	Acontece quando apenas uma role da relação tem conhecimento da outra. Por exemplo, funcionario e endereço. O funcionario tem um referencia para seu endereço, mas não existe a necessidade do endereço saber quem mora ali:

	[FUNCIONARIO] -----> [ENDERECO]

	Bidirecional:
	Acontece quando as duas roles da relação se conhecem e uma pode acessar a outra. Por exemplo, na relação entre projeto e funcionarios, o funcionario precisa saber em qual projeto esta trabalhando e o projeto precisa poder consultar quem trabalha nele:

	[FUNCIONARIO] <-----> [PROJETO]


	* CARDINALIDADE

	É a relação "quantos para quantos" que existe entre as roles. Podem ser:

	1) Um para um
	[FUNCIONARIO] 1 -----> 1 [ENDERECO]
	Um funcionario tem um endereço;
	Um endereço tem um funcionario (não é necessariamente verdade, mas ok)

	2) Um para muitos
	[FUNCIONARIO] * ------> 1 [DEPARTAMENTO]
	Um funcionario trabalha em apenas departamento;
	Um departamento tem varios funcionarios;

	3) Muitos para um:
	

	4) Muitos para muitos
	[FUNCIONARIO] * <-------------> * [PROJETO]
	Um funcionario pode estar em varios projetos;
	Um projeto pode ser composto de varios funcionarios;

	* ORDINALIEDADE

	Define se a entidade target é mandatoria ou não no momento de persistir a entidade source.
	Caso seja mandatorio e a entidade target não exista, a entidade source esta em um estado invalido;
	Representado através do intervalor "0..1" ou "0..*" a anotação de cardinalidade;

	Exemplo:
	[FUNCIONARIO] 0..* ------> 1 [DEPARTAMENTO]


* Relacionamentos - Mapeamento


	Basico de relacionamento

```
	 _________				 _____________
	|Employee |			 	| Departament |
	|_________|			 	|_____________|	
	| id	  |				| id 	      |
	| name    | ---------->	                | name        |
	| salary  |                             |             |  
	| depto_id|				|	      |
	|_________|				|_____________|

```

```
         _______________________________________________________
	|Roles: 			Employee e Departament |
	__________________________|____________________________|
	|Soure: 		        Employee
	__________________________|____________________________|
	|Target: 			Departament
	__________________________|____________________________|
	|Join Column:	                "depto_id" em Employee;
	__________________________|____________________________|
	|Owner do relacionamento: 	Employee
	__________________________|____________________________|
	|Inverse Side: 			Departament  
	__________________________|____________________________|
```

	**********************************************************************
	As propriedades e caracteristicas do mapeamento é sempre definido no lado dono da chave.

	In almost every relationship, independent of source and target sides, one of the two sides will have the join column in its table. That side is called the owning side or the owner of the relationship. The side that does not have the join column is called the non-owning or inverse side.
	**********************************************************************


	* @ManyToOne

	Um relacionamento "muitos para um" é definido através da anotação @ManyToOne no atributo do tipo do target(Departament) na entidade source (Employee):  

	UML:

```
	 _________				 _____________
	|Employe  |			 	| Departament |
	|_________|			 	|_____________|	
	| id	  |				| id 	      |
	| name    | ---------->	                | name 	      |
	| salary  |                             |-------------|  
	| depto_id|				
	|_________|				

```



	Classe: 

```
	@Entity
	public class Employee {
            // ...
            @ManyToOne
            private Department department;
            // ...
	}
```
	@JoinCOlumn

	Anotação usada para definir qual será o campo da foreign key do relacionamento.
	No exemplo acima, do campo 'department' da entidade Employee, deve ser anotado com @JoinCOlumn. Dessa forma, a chave primaria de Departament será usada como chave estrangeira.
	A chave sera definida de qualquer forma em Employee. A anotação @JoinColumn só sera usada para customizar esse campo na tabela (nome, tipo, etc)
	Essa anotação vai sempre no lado dono do relacionamento.



	* OneToOne

	Relacionamento em que a cardinalidade é de apenas um ambos os lados

```
	 _________				    _____________
	|Employe  |			 	    | ParkingSpace|
	|_________|			 	    |_____________|	
	| id	  |				    | id 	  |
	| name    | 1 --> 1                    | lot         |
	| salary  |                                 | location    |  
	| depto_id|				    |	 	  |
	|_________|				    |_____________|

```

	Entidades 
	
```
@Entity
@Table(name = "employee")
public class Employee {
		
    @OneToOne
    @JoinColumn
    private ParkingSpace parkingSpace;
}
                

@Entity(name="parking_space")
public class ParkingSpace {

    @OneToOne(mappedBy="parkingSpace")
    private Employee employee;

}
```

	The two rules, then, for bidirectional one-to-one associations are the following:

		The @JoinColumn annotation goes on the mapping of the entity that is mapped to the table containing the join column, or the owner of the relationship. This might be on either side of the association.

		The mappedBy element should be specified in the @OneToOne annotation in the entity that does not define a join column, or the inverse side of the relationship.
	---------------------------------------------------------------------------------------------

	MappedBy

		@Xpto(mappedBy="XPTO")

		O valor do atributo mappedBy deve ser o nome do atributo (parkingSpace) na entidade owner (Employee) do relacionamento que aponta de volta para a entidade inversa do relacionamento (ParkingSpace)


		Na falta de mappedBy e JoinColumn nas duas entidades do relacionamento, ambos os lados serão considerados os owners do relactioamento, tendo assim cada uma chave estrangeira para a outra entidade.

		Coisas estranhas acontece quando não se usa o "mappedBy";
			Serve basicamente para informar a entidade que esse é um relacionamento bidirectional. Sem isso, ambos os lados do relacionamento irão considerar que estão em um relacionamento unidirecional.


	* OneToMany

	OneToMany(mappedBy = )
		Todo relactioamento bidirectional OneToMany implica em um relacionamento ManyToOne no outro lado.
	
	Falto do mappedBy e a tabela de junção 
		Pensando na tabela Departament, como colocar diversas chaves estrangeiras referente a cada funcionario que trabalha nesse departamento em uma só tabela? Impossivel!
		Então, para fazer isso, na ausencia do parametro mappedBy, informando que esse lado do relacionamento não é o lado dominante e será criada uma tabela de junção.

		Tabela departament_employee.
		Campos:
				CAMPO 			NULLABLE    CHAVE   	
			staff_id		| 	NO	 	 |	PRI 	|	
			departament_id	| 	NO		 |  MUL 	|

	
	
	EntityTarget e Collection vs Collection<T>
		Caso a collection utilizada para adicionar os campos não use generics, pode ser usado o campo entityTarget ta anotação @xptoToMany

	***************************************************************************************************
	The many-to-one side should be the owning side, so the join column should be defined on that side.
	The one-to-many mapping should be the inverse side, so the mappedBy element should be used.
	***************************************************************************************************


	* ManyToMany

		Tabela de relacionamento
		Ao usar esse tipo de relacionamento, sera obrigatoriamente criado uma (ou duas!!) tabela relacionamento entre as duas entidades.
		Essa tabela terá duas colunas, ambas chaves estrangeiras que formaram a uma chave primaria, cada uma apontando um entrada de uma das tabelas compoentes do relacionamento.
		Para personalizar a tabela criada, é possivel usar a anotação @JoinTable.

			public class Employee {
				@ManyToMany
		    	@JoinTable(name = "employee_project",
		            joinColumns = {@JoinColumn(name = "emp_id")}, 
		            inverseJoinColumns = {@JoinColumn(name = "proj_id")})
		    	private Collection<Project> projects;
	    	}

	    	Será criada uma tabela employee_project (por acaso o mesmo nome que o nome padrão), a coluna representado a chave primaria da tabela dona do relacionamento será "emp_id" e a coluna representado a tabela Project, será "proj_id"


		Importancia do mappedBy em um dos lados
		Nesse relacionamento, nenhum dos lados é explicitamente o owner do relacionamento. Então, não é necessario utilizar a anotação @JoinColumn, porque será criado uma tabela para fazer essa união. Porem, entretanto, todavia, caso um dos lados não utilize o campo mappedBy, serão criados duas tabelas para esse relacionamento.
		Por exemplo, no relacioanemto muitos para muitos entre Employee e Projects:

		@JoinTable


		public class Employee {

			[...]
			
			@ManyToMany
			private Collection<Project> projects;
		}

		******
		
		@Entity(name = "project")
		public class Project {
		    
		   @ManyToMany(mappedBy="projects")
		   private Collection<Employee> team;
		}

		Caso uma das entidades não tenha o atributo mappedBy definido, serão criadas as tabelas employee_project e project_employee!
		No exemplo acima, como Project se declara como o lado inverso do relacionamento (mappedBy), será criado apenas a tabela employee_project. Tabela essa "customizada" pela anotação @JoinTable.


	* Eager e Lazy

		Para valores simples
		Para collections
		É só um dica

	* Embedable

		O que é?
		Certas 'entidades' no mundo OO, fazem sentido estarem em classes separadas, porem no mundo relacional (banco de dados), as vezes nao faz sentido que estejam em tabelas separadas. Assim, é possivel definir que um atributo em uma classe, ao ser criado o banco de dados, não deve ser uma chave estretegia referenciando para outra entidade. Em vez disso, os atributo dessa outra entidade, serão adicionados na tabela da entidade primaria.

		Exemplo:

			Pense em cidade e estado dentro do endereço. No mundo OO, faz sentido estado e cidade estar em uma classe (chamada de Localização, por exemplo) separada e rua e numero (chamada de Endereço, por exemplo). E Endereço ter uma referencia para Localização.

			Porem, ao ser gerada as respectivas tabelas, os campos cidade e estado, devem ser criados na tabela endereço e não em uma nova tabela chamada Localização;


			 _________	
			|Endereco |
			|_________|
			| rua	  |
			| bairro  |
			| numero  |
			| cidade  |
			| estado  |
			|_________|


		Como definir uma classe embedable?
			A classe não deve ser anotada com @Entity e não deve ter um @Id. Em vez disso, a mesma deve ser anotada com @Embedable;

		Como definir um atributo embedado?
			Na classe que irá embedar essa classe, opicionalmente, esse atributo pode ser anotado com @Embedade

		Exemplo:

		Classe que será embedada

			@Embeddable
			public class Location {
			    
			    private String city;
			    
			    private String state;
			}


			@Entity(name = "address")
			public class Address {

				[...]
			   	@Embedded
				private Location location;
				[...]
			}	
		

		Sobreescrevendo atributos

			Por algum motivo, talvez vc queira mudar alguma informação sobre os atributos da classe que será embedada. Para isso, use as anotações AttrubuteOverride e AttrubuteOverrides


			Por exemplo, para mudar os nomes dos campos 'city' e 'state' para 'cidade' e 'estado'

			@Entity(name = "address")
			public class Address {

			[...]

			    @Embedded
			    @AttributeOverrides({ 
			                @AttributeOverride(name = "state", column = @Column(name = "estado")),
			                @AttributeOverride(name = "city", column=@Column(name="cidade"))
			             })
			    private Location location;
			[...]
		 }

		 	A anotação @Column tem diversos parametros que podem ser usados para definir a nova coluna.



************************************************* CHAPTER 5 - Collection Mapping ************************************************* 

	Coleção de entidades, embedables e tipos basicos;
	@ElementCollection
	@CollectionTable

	Referencias para collections com interfaces e nao implementações
		Porque ao tornar a lista um objeto gerenciado pelo provider, o mesmo pode trocar a implementacao por uma propria.

	
	@OrderBy
		ordenacao baseada em um atributo da entidade, no momento que a mesma esta na memoria. Ordenacao feita somente ao ler os elementos, e não altera a posicao das linhas na base de dados.



		Exemplo:

			Ordenando uma lista de elementos embedados:

				@OrderBy("data_entrada DESC, placa")
				@ElementCollection
				private List<Carros> patio;

			Ordenando os funcionarios por nome:

				public class Department {
				    // ...
				    @OneToMany(mappedBy="department")
				    @OrderBy("name ASC")
				    private List<Employee> employees;
				    // ...
				}

		Para ordernar por elemento de uma classe embedada na entidade, use o "."
		Exemplo:

		A classe Address tem embedado a classe Location. Para ordernar endereços (Address) por um campo de Location, faça assim:

		@OneToMany
		@OrderBy("location.city")
    	private List<Address> address; 

		falta dessa anotacao em:
			colecao de entidades: Ordenação ascendente pela primary key
			colecao de embebeddos: Ordenacao indefinida. Forma devolvida pelo banco de dados.
			colecao de elementos basicos: Ordenacao natural dos itens.




	@OrderColumn
		Utilizado quando a entidade ou elemento basico nao tem nenhum campo que pode ser usado como ordenador, como seria o caso de @OrderBy.
		Ao usar essa anotacao, um campo é criado na tabela e o mesmo será usado para persistir a ordem da lista.

		Onde vai?

			Ao contrario das outras anotações que vao no lado dono do relacionamento, essa anotacao vai no lado inverso, onde fica a lista, MAS o campo  ('PRINT_ORDER'), será criado no lado dono do relacionamento.

			public class PrintQueue {

				@OneToMany(mappedBy="queue")
			    @OrderColumn(name="PRINT_ORDER")
			    private List<PrintJob> jobs;
			}


		Nome padrão
			Todos os campos da anotação são opcionais. Caso o nome nao fosse informado, seria gerada uma coluna com o nome do campo, concatenado com ORDER. No exemplo acima, seria "jobs_ORDER"

		Um ponto de atenção é que ao executar uma alteracao na ordem da lista persistinda, um UPDATE será executado para que os indices guardados sejam atualizados.



	Maps como coleções

		Permutacao de valores chave e valor: entidade, embedable e elementos basicos;

		Mapeamento:

			É sempre value do Map que define o tipo de anotação usada para o mapeamento:
			Map<X,Basic Type ou Embedable>: @ElementCollection
			Map<X,Entidade>: @OneToMany ou ManyToMany

		Cadê  key que estava aqui?


			<BasciType,Element> 

				Se for uma Element e Key for um Basic type, seu valor será armazeado na CollectionTable que será criada.
				Por exemplo:


				public class Employee {

				   	@ElementCollection
				    @CollectionTable(name="emp_phone", joinColumns=@JoinColumn(name="employee_id"))
				    @MapKeyColumn(name="phone_type")	
				    @Column(name="phone_number")
				    private Map<String, String> phoneNumbers;
				}

				Será mapeado como:

				Tabela gerada: emp_phone
				Mapeamento da classe source: Será o id do funcionario, guardado na tabela e com o nome "employee_id" (@JoinColumn)
				Mapeamento da chave do mapa: Será guardado na tabela gerada ("emp_phone") com o nome "phone_type" (@MapKeyColumn)
				Mapeamento do valor do mapa: Tambem será guardado na tabela gerada ("emp_phone") na coluna "phone_number" (@Column)


				O mapeamento da chave do funcionario será uma PK e uma FK e a chave do mapa (phone_type), será uma PK. Assim, não pode haver um mesmo funcionario com dois telefones celulares cadastrados para si, por exemplo.

				Para evitar erros de digitação ou tipos invalidos, a chave deveria ser uma ENUM. Ao persistir uma enum, JPA persiste o valor ordinal da enum e não seu valor (Aaaarrrg!). Para alterar isso, fazendo com que seja salvo o valor da enum, use a anotação @MapKeyEnumerated. 

					{

						@ElementCollection
					    @CollectionTable(name="emp_phone", joinColumns=@JoinColumn(name="employee_id"))
					    @MapKeyColumn(name="phone_type")	
					    @Column(name="phone_number")
					    @MapKeyEnumerate
					    private Map<PhoneType, String> phoneNumbers;
				    }

				Existe ainda a anotação  @MapKeyTemporal para quando a chave do mapa é um Date. Os valores dessa anotação são DATE, TIME ou TIMESTAMPA, definindo em qual formato a data será persistida.


			<BasciType,Entity>

			 Pense em mapear um funcinoario por sua posição no departamento
			 Na entidade Departament, haveria um Map onde  chave seria o id do cubiculo do funcionario e a chave seria a entidade do Employee;
			 Ao fazer o mapeamento dessa forma, a tabela Employee, alem de ter o id do departamento a qual pertence, teria ainda o id do seu cubiculo nesse respectivo departamento.

			 O mapeamento da entidade:

			 	public Departament() {

					/** Forma antiga
					OneToMany(mappedBy = "department", targetEntity = Employee.class)
					private List<Employee> staff;
					*/

				    @OneToMany
				    @MapKeyColumn(name = "cub_id")
				    @JoinTable(name = "dept_emp", 
				        joinColumns = @JoinColumn(name = "departament_id") ,
				        inverseJoinColumns=@JoinColumn(name="employee_id")
				    )    
				    private Map<String, Employee> employeeByCubicle;

				}


				A chave do departamento e do cubiculo nessa tabela seriam chaves primarias e o id do funcionario seria uma unique constraint;

			Keying by Entity Attribute

				As vezes pode ser necessario criar um relacionamento, que caberia uma lista, dentro de um Map. Por exemplo Departament e seus (lista) Employees. Pode ser necessario que ao inves de uma lista de funcionario seja utiliado um mapa em que a chave seja o id do funcionario. Para isso, use a anotacao @MapKey

				@Entity(name = "departament")
				public class Departament {

					@OneToMany(mappedBy = "department", targetEntity = Employee.class)
					@MapKey(/*name="id"*/)
					private Map<Integer,Employee> staff;
				}

				A tabela gerada no banco de dados nao muda em nada. Ao omitir o nome (que se refere ao nome de um atributo da target entity) da anotacao @MapKey, por padrão será usado o atributo ID da entity target


************************************************* CHAPTER 6 - Entity Manager ************************************************* 


	Container-Managed Entity Managers
		

		Testes:
			Em um metodo de um Session Bean Stateless recuperar a transação que é automaticamente criada ao entrar no metodo e dar um commit nela.


		Esse tipo de entity manager somente existe no mundo EE, por ser criado e gerenciado pelo container (Wildfly, WAS, etc).
		É criado pela aplicação ao utilizar a annotation @PersistenceContext.
		O container cuida de todo o ciclo de vida desse cara. 
		Esse entity manager se divide em dois tipos:
		
		Transaction-Scoped Persistence context

				Stateless
				É em sua natureza stateless. Quando um metodo é iniciado, é verificado se, attached a transação corrente ja existe um persistence context associado. Caso não, um será criado e as entidades do metodo serão associado a ele. No fim do metodo, a transação será comitada ou sofrerá rollback, e com isso, o persistence context deixa de existir.

				Metodos EJB sempre em uma transação
				
				Alterações na entidade refletidas na tabela
					Como as entidades  estão associadas a um contexto, qualquer mudança feitas nelas, será refletido no banco de dados quando a transação for comitada.


		Extended Persistence context 

				Diferente do Transaction-Scoped, esse persitence context tem um ciclo de vida que dura alem da transação.
				Para utiliza-lo, é necessario dizer isso explicitamente para o container. Para isso, defina a anotação type da anotação @PersistenceContext:

					@PersistenceContext(unitName="EmployeeService", type=PersistenceContextType.EXTENDED)
				

				Statefull Sesisson Bean

					DEVE ser utilizado com statefull session beans.

				Ciclo de vida do persistence context

					Como dito, esse persistence context tem uma vida longa, diferente do Transaction-Scoped. Ao definir a injeção de um desses em um statefull session bean (type=PersistenceContextType.EXTENDED), quando esse bean for criado, juntamente com ele tambem será criado seu contexto.
					Esse contexto vivera entre as chamadas dos metodos. Uma referencia para uma entidade que fique salva nesse bean nunca ficará em estado detached (a não ser que feito explicitamente).
					Assim como o bean, esse contexto morrerá quando for chamado um metodo anotado com @Remove.


	
	Application-Managed Entity Managers

		Por padrão, o persistence context associado a esse entity manager terá o escopo extended. Só termina quando o metodo close é chamado no entity manager.

		Criação

			No mundo SE:

				Para criar um desses tipos de entity manager (o unico disponivel para esse mundo), faça assim:

					EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("PersistenceUnit");
        			EntityManager entityManager = entityManagerFactory.createEntityManager();

        		O nome da persistence unit é o mesmo configurado no persitence.xml.
        		Necessario chamar o metodo close no EntityManagerFactory e no EntityManager

        	No mundo EE:

        		Ao inves de injetar diferente um EntityManager (@PersistenceContext), injete um EntityManagerFactory:

        			@PersistenceUnit(name="persitence.xml")
        			private EntityManagerFactory factory;

        			public void meuMetodoDeNegocio(){
        				EntityManager em =  factory.createEntityManager();
        				em.persiste(...)
        			}

        		Necessario chamar o metodo close no EntityManager.
        		O EntityManagerFactory ainda é gerenciado pelo container.

        	Ao criar um EntityManagerFactory, alem do metodo que recebe uma String com o nome da persistence unit cadastrada no persistence.xml, existe uma versão sobrecarregada que recebe um Map. Esse objeto pode ser usado para sobrescrer ou adicionar atributos na criação da factory. Util isso para situações em que alguma propridade de conexão só existe em tempo de execução.

		EntityManager - EntityManagerFactory [Close]


		 
	Transaction Management


		JTA-Transaction Managed

			Explicar a relação EntityManager, Transaction e Persistence Context

				Um entity manager "gerencia" um persistence context;
				Esse persitence context esta amarrado a transacao corrent.


			Java SE vs EE
				SE: Somente Resource-local
				EE: Resource-local e JTA, o ultimo sendo o default e preferencial.

			JTA Transaction Management

				Transaction association
					Ato de asssociar o persistence context com a transaçao corrente.
					Apenas um persistence context associado a transação corrente.

		  		Transaction synchronization
		  			Eu não vejo diferença entre esse e o de cima.

		  		Transaction propagation
		  			Diferentes beans podem ter diferentes instancias do entity manager. Porem quando o entity manager que é chamado dentro de uma transação, antes do mesmo associar um novo peristence context com a transação corrente, ele verifica se ja existe um associado. Se sim (transaction propagation), ele usa esse mesmo pesistence context que já esta associado com a transação propagada entre metodos/beans


		  	Transaction-Scoped Persistence Contexts

		  		Amarrado a transacao

		  			Quando um metodo é invocado no entityManager, o mesmo ira verificar se existe um persiste context atrelado a transaction corrente. Caso exista, esse contexto será utilizado. Caso não, um será criado e atrelado a transacao.
		  			Nos metodos seguintes que FOREM EXECUTADOS DENTRO DESSA MESMA TRANSACAO (por exemplo, um metodo que tiver @REQUIRED_NEW, usara outro persistence context, por ter outra transação), o persistence context já ira existir e será utilizado.
		  			Com isso, entidades alterados, criadas, deletadas, mas que ainda não foram persistidas no banco, devido a transacao dessa ação ainda não ter sido comitada, estaram com seus valores atualizados dentro do persistence context.

		  		Lazy initiation do persistence context

		  			Ao entrar dentro de um metodo que ira rodar dentro de uma sessão, o persistence context não será criado e atrelado a transação até que um metodo no entity manager seja chamado.

				When a method is invoked on the transaction-scoped entity manager, it must first see whether there is a propagated persistence context 

					find, persist, update, delete, etc:
						Quando metodos do entityManager são chamados, o entity manager deve primeiro checar se um peristence context ja esta existe e esta sendo propagado para esse chamada.


		  		The rule of thumb for persistence context propagation is that the persistence context propagates as the JTA transaction propagates
		  			
		  			Regra: enquanto a transação se propagar, o persistence context associado a ela tambem irá ser propagado.
		  				ou 'seje',
		  					varios metodos -> mesma transação -> mesmo persistence context.

		  		[DUVIDA]
		  			Ja que um Extended Persistence Context  é iniciado de forma preventiva assim que o Stateful Session Bean é instanciado, o que acontece quando um contexto como esse é chamado por Stateless Session Bean (Transaction-Scoped)??

		  			[RESPOSTA]:
		  				Caused by: javax.ejb.EJBException: WFLYJPA0030: Found extended persistence context in SFSB invocation call stack but that cannot be used because the transaction already has a transactional context associated with it.
		  				Para "resolver" a situação (Stateful Session Bean) pelo menos é chamado, é possivel anotar o metodo deele com @TransactionAttribute(TransactionAttributeType.REQUIRES_NEW). Porem, como é uma nova transação, beans alterados/adicionados/removidos pelo metodo que o chamou, não estarão atualizados nesse persistence-context
							  			
		  		Teste:
		  			1
			  			Criar uma transacao (CMT ou BMT? JTA ou Resource-local?)
			  			Salvar um Object
			  			Entrar em um metodo
			  			[SIM|NAO]Iniciar outra transacao
			  			Recuperar o objeto criado antes

			  		2
			  			Criar uma transacao
		  				Alterar o valor de um objeto na base
		  				Entrar em outro metodo
		  				[SIM|NAO]Iniciar outra transacao
		  				Recuperar esse mesmo objeto alterado
		  				Checar o valor

		  	Extended Persistence Contexts

		  		Eager association

		  			Como o persiste context já irá existir antes da chamada de algum metodo que será executado em contexto de transação, o container, de modo eager, ao entrar nesse metodo, irá associar o extended persistence context a transação corrente. Dessa forma, não será criado um novo.

		  		Amarrado ao statefull session bean

		  			O ciclo de vida de um extended persistence context inicia quando o statefull session bean é iniciado é só é 'morto', quando esse bean é removido (@Remove)

		  		Sharing peristence context com Transaction-Scoped persistence context

		  			Caso um statefull session bean com extended persitence context chame algum metodo em um bean que use um Transaction-Scoped persistence context, esse ultimo utilizara o mesmo contexto propagado pelo extended peristence context.


		  		Duvida:
		  		Em um extended persistence context, quando a transação é comitada? No fim de todo metodo ou somente ao executar o metodo @Remove?

		  		Stateful bean usually use extended persistence context. The persistence is created when the bean instance is created and only destroy when the stateful bean is removed. The extended persistence context will be associated to a transaction when a method of stateful bean is called. When the method is over, transaction is gone, but the persistence context does not  go with the transaction but stay for the next transaction until the whole stateful bean is removed by the container.

		  	
		  	Persistence Context Collision

		  		Somente um contexto ativo 
		  		Transaction-Scoped -> Extended (NOT!)
		  		Extended -> Transaction-Scoped (OK)
		  		Extended -> Extended (OK)



		  		********************************************************************************************************************************
		  		In this article, for simplicity, when we mention stateless bean, we mean a stateless bean with EntityManager variable and its methods operate on the persistence

		  			Porque pode ser que o stateless bean nem tenha uma instancia do EntityManager e mesmo que tenha, caso um de seu(s) metodo(s) não chame(m) algo no EntityManager, um persistence context nao será criado e ai blz chamar um statefull session bean com extended persistence context.
		  		********************************************************************************************************************************

		  		REQUIRES_NEW
		  		NOT_SUPPORTED

		  		Duvidas:
		  			2 metodos com REQUIRED_NEW dentro do mesmo statefull session bean. Quando um chamar o outro, 2 transactions serão criadas? Mesmo dentro do mesmo bean?

		  			Note: Since this mechanism is based on proxies, only 'external' method calls coming in through the proxy will be intercepted. This means that 'self-invocation', i.e. a method within the target object calling some other method of the target object, will not lead to an actual transaction at runtime even if the invoked method is marked with @Transactional!


		  		Persistence Context Inheritance

			  		@Stateful
					public class MyStatefulService {

					    @EJB
					    FirstDAO firstDAO;

					    @EJB
					    SecondDAO secondDAO;

					    public void doSomething() {
					        // Fail!
					        assertTrue(firstDAO.loadItem() == secondDAO.loadItem());
					    }

					    @Remove
					    public void endConversation() {
					    }
					}


					@Stateful
					public class FirstDAO/SecondDAO {

					    @PersistenceContext(type = PersistenceContextType.EXTENDED)
					    EntityManager em;

					    public Item loadItem() {
					        return em.find(Item.class, 1);
					    }
					}



		Application-Managed Persistence Contexts

			Criados atraves de uma instancia do EntityManagerFactory.

			An application-managed entity manager participates in a JTA transaction in one of two ways. If the persistence context is created inside the transaction, the persistence provider will automatically synchronize the persistence context with the transaction. If the persistence context was created earlier (outside of a transaction or in a transaction that has since ended), the persistence context can be manually synchronized with the transaction by calling joinTransaction()on the EntityManager interface. Once synchronized, the persistence context will automatically be flushed when the transaction commits.


			[SITUAÇÃO 1]
				O EntityManager é criado dentro de um metodo que já tem a transação ativa.
				Nesse caso, o mesmo sera assoaciado a transação corrente e quando a mesma for encerrada, o contexto sera comitado.


			Exemplo:

				@Stateful
				public class DepartmentManager {
					
					@PersistenceUnit(unitName="EmployeeService")
				    EntityManagerFactory emf;
				    
				    Department dept;

				    //TRANSAÇÃO 1
				    public void init(int deptId) {
				        EntityManager em = emf.createEntityManager();
				        dept = em.find(Department.class, deptId);
				    }

				    public void addEmployee(int empId) {

				    	EntityManager em = emf.createEntityManager();
				    	Employee emp = em.find(Employee.class, empId);
				        dept.getEmployees().add(emp);
				        emp.setDepartment(dept);
				    }
				}



			[SITUAÇÃO 2] 
				O EntityManager é criado antes de existir uma transacao ativa.
				Nesse caso, é necessario informar ao entity manager que ele deve sincronizar com a transacao corrente

			@Stateful
			public class DepartmentManager {
			    @PersistenceUnit(unitName="EmployeeService")
			    EntityManagerFactory emf;
			    EntityManager em;
			    Department dept;

			    //TRANSAÇÃO 1
			    public void init(int deptId) {
			        em = emf.createEntityManager();
			        dept = em.find(Department.class, deptId);
			    }

			    //TRANSAÇÃO 2
			    public void addEmployee(int empId) {
			    	em.joinTransaction();
			        Employee emp = em.find(Employee.class, empId);
			        dept.getEmployees().add(emp);
			        emp.setDepartment(dept);
			    }

			    Like the container-managed extended persistence context, the Department entity remains managed after the call to init
			    When addEmployee is called, there is the extra step of calling joinTransaction to notify the persistence context that it should synchronize itself with the current JTA transaction. Without this call, the changes to the department would not be flushed to the database when the transaction commits.

			    Application-Managed PersistenceContext não propagam. Igual o Session do Hibernate
			    
			    [DUVIDA]Qual a diferença de um que propaga e um que não.
					[RESPOSTA]Because application-managed entity managers do not propagate, the only way to share managed entities with other components is to share the EntityManager instance.

			application code must not call joinTransaction on the same entity manager in multiple concurrent transactions.

			 The first is that application-managed entity managers are automatically synchronized with the transaction if they are created when a transaction is active. 


		Unsynchronized Persistence Contexts

			Diferença do padrão
			Por que não usar um application-managed?
				Porque meu caro rapaz, sendo um application managed persistence context, vc perderia algumas capacidades de container-managed, como propagação.

			//Container managed
			@PersistenceContext(unitName="EmployeeService", synchronization=UNSYNCHRONIZED)
			EntityManager em;

			//Application managed
			@PersistenceUnit(unitName="EmployeeService")
			EntityManagerFactory emf;
			...
			EntityManager em = emf.createEntityManager(UNSYNCHRONIZED);

			Para acessar a transação: joinTransaction
			SOmente a primeira transacao. Isso é: esse entity manager esta se associando a transacao corrente. Será necessario chamar o joinTransaction em todas as proximas transacões que voce desejar que o entity manager seja sincronizado.

		
		Resource-Local Transactions || EntityManager

			EntityManager.getTranscation
			EntityTransaction vs UserTransaction
			Resource-Local Transactions não rollback se uma JTA transaction dar merda;
			Resource-Local Transactions podem ser abertas dentro de JTA transactions;		

			There are only six methods on the EntityTransaction interface. The begin method starts a new resource transaction. If a transaction is active, isActivewill return true. Attempting to start a new transaction while a transaction is active will result in an IllegalStateException being thrown. Once active, the transaction can be committed by invoking commit or rolled back by invoking rollback. Both operations will fail with an IllegalStateException if there is no active transaction. A PersistenceException will be thrown if an error occurs during rollback, while a RollbackException, a PersistenceException subclass, will be thrown to indicate the transaction has rolled back because of a commit failure.
			If a persistence operation fails while an EntityTransaction is active, the provider will mark it for rollback. It is the application’s responsibility to ensure that the rollback actually occurs by calling rollback(). If the transaction is marked for rollback, and a commit is attempted, a RollbackException will be thrown. To avoid this exception, the getRollbackOnly method can be called to determine whether the transaction is in a failed state. Until the transaction is rolled back, it is still active and will cause any subsequent commit or begin operation to fail.

		Transaction Rollback and Entity State

			Quando um rollback acontece:
				rollback no banco de dados
				persiste context é limpo


			Cuidados a ser tomar ao tenta persistir uma detached entity
				Limpe as primary keys. Explique.



	Entity Manager Operations

		Persisting an Entity
			já é salvo? 
				[NAO. Apenas passa a ser gerenciado pelo persistence context e mudanças futuras [dentro do mesmo persistence context [geralmente dentro da mesma transcao]] serao salvas]
			Passa a ser gerenciado.
			Sem transação ativa: 
				Transaction-Scoped: TransactionRequiredExceptio
				 Extended e Application-Managed: A requisicao para gerenciar essa entity sera enfileirada.

			Somente para novas entidades
				Caso seja chamado em uma entidade que ja esta no persistence contex, é ignorado.
				Caso a entidade nao exista no persistence context, mas ja exista no banco de dados, sera lançado uma EntityExistsException.

			Entidade duplicada

		Finding an Entity
			getReference, o proxy e o Id
			EntityNotFoundException


		Removing an Entity
			O problema com relacionamentos.

				Ao remover uma entidade, é necessario tomar cuidado com as constraints do banco de dados.

				No exemplo abaixo, na tabela Empregado existe uma chave estrangeira para a tabela ParkingSpace. Ao remover a vaga, sem antes remover a associação da tabela Empregado, será lançado uma exception;

				[Employee] -> [ParkingSpace]
				Employee emp = em.find(Employee.class, empId);
				em.remove(emp.getParkingSpace());

				O correto seria primeiro remover a associação e depois remover a entidade target.

				Employee emp = em.find(Employee.class, empId);
				ParkingSpace ps = emp.getParkingSpace();
				//O registo em Empregado nao aponta mais para ParkingSpace
				emp.setParkingSpace(null);
				em.remove(ps);


			Cascading Operations

				Define quando/qual operação feita em uma entidade será cascateada para seus relacionamentos.
				Por padrão, isso nunca acontece.

				Para algumas operações, faz sentido que a mesma nao seja propagada. Por exemplo, ao remover uma entidade, geralmente deseja-se apenas remover a mesma. Ja ao persistir uma entidade, talvez seja desejavel que seus relacionamentos ainda nao persistidos, assim sejam.

				Unidirecional
					Operacoes em Cascade sao unidirecionais. Ou seja:

					@Entity
					public class Employee {
					    // ...
					    @OneToOne(cascade={CascadeType.PERSIST, CascadeType.REMOVE})
					    ParkingSpace parkingSpace;
					}

					Caso eu persista o Empregado, sua vaga sera persitida. MAS o contrario nao é verdadeiro. Se eu persistir a vaga, o Empregado nao sera persistido automaticamete.

				Cascade Persist
					Como dito antes, se uma entidade ja existe no persitence context, caso o metodo persiste seja chamado passando o mesmo objeto, o mesmo sera ignorado. MAS, entretanto, toda vida, pore, caso o mesmo tenha um novo relacionamento que este com CascadeType. PERSIST, essa nova entidade sera adicionada ao persitence context.

				Cascade Remove
					 makes sense: one-to-one and one-to-many relationships
					 Care must be taken when using the REMOVE cascade option.
					 the link between the two instances remains


			Clearing the Persistence Context

					O que acontece com persistence context?

						Todas as entidades que haviam sido adicionadas ao persistece context serão removidas e caso a transaction seja comitada, "nada acontece, feijoada", porque o persistence esta vazio.
						Cuidado ao chamar EntityManager.clear quando existem mudanças nao comitadas. Nao faz muito sentido.



	Synchronization with the Database

		EntityManager.flush
		O que faz?
		Flush ao executar uma query
		Flush de uma entidade detatched
		Flus: Persist e Remove
		Flush: Persistindo uma entidade que tem atributos não gerenciados e sem o cascade PERSIST [IllegalStateException]


	Detachment and Merging

		Merge: Any changes to entity state that were made on the detached entity overwrite the current values in the persistence context
		
			Teste: Find em uma entidade que tem uma lista carregada em modo lazy, detach a entidade e ver como estão os valores da lista.
			Resposta: The behavior of accessing an unloaded attribute when the entity is detached is not defined. Some vendors might attempt to resolve the relationship, while others might simply throw an exception or leave the attribute uninitialized. 


		EntityManager.merge
			Sincroniza o objeto passado com o persistence context
			Retorna uma referencia de um objeto gerenciado. O objeto passado como parametro não gerenciado e sim o retornado. (Mais ou menos igual trabalar com Strings)

			public void updateEmployee(Employee emp) {
			    em.merge(emp); // Vai ser atualizado no banco de dados
			    emp.setLastAccessTime(new Date()); //Não vai ser atualizado no banco de dados
			}

			public void updateEmployee(Employee emp) {
			    Employee managedEmp = em.merge(emp); // Vai ser atualizado no banco de dados
			    managedEmp.setLastAccessTime(new Date()); // Vai ser atualizado no banco de dados
			}

		Avoiding Detachment
			Transaction View
				Conceito que parte do principio de deixar com que o controller da view [Servlet no caso] gerencie a sessão. Ele abre uma sessão antes de fazer a criação do entityManager, o qual quando for criado terá seu persistence context adicioado a transação criada, e o proprio controller fecha o entity manager, após ter feito o dispatch da requisicao para o view,




========================= CHAPTER 7 - Using Queries ================================================================================================

Java Persistence Query Language
	
	Duvidas:
		SELECT e.name, e.salary FROM Employee e
		Qual o tipo retornado?

		Um array de object, sendo a primeira posição uma String referente a name e um Double, referente ao salario.



	JOIN 
		SELECT p.number
		FROM Employee e, Phone p
		WHERE e = p.employee AND
		      e.department.name = 'NA42' AND
		      p.type = 'Cell'

		SELECT p.number
		FROM Employee e JOIN e.phones p
		WHERE e.department.name = 'NA42' AND
		      p.type = 'Cell'


	NamesQuery tem a vantagem de poderem ser processadas de antemão, quando a aplicação sobe. Custo de processamento apenas uma vez.

	An issue to consider with string dynamic queries, however, is the cost of translating the JP QL string to SQL for execution. A typical query engine will have to parse the JP QL string into a syntax tree, get the object-relational mapping metadata for each entity in each expression, and then generate the equiva 	ent SQL. For applications that issue many queries, the performance cost of dynamic query processing can become an issue.


	NamedQuery estao disponiveis no contexto no persistence unit. Todos os persistences context tem acesso a elas.

	Caso em runtime perceba-se que uma Query passou a ser muito utilizada e deseja-se remover seu custo de processamento, a mesma pode ser adicinada como uma NamesQuery, que será prcessada apenas uma vez:

	private String myQuerySinistra = "";

	@PersistenceContext(unitName="QueriesUnit")
    EntityManager em;

	public void init(){
		TypedQuery<Long> q = em.createQuery(QUERY, Long.class);
       	em.getEntityManagerFactory().addNamedQuery("findSalaryForNameAndDepartment", q);
	}


	Ao usar Date ou Calendar como parametros de um query, é necessario informar um terceiro parametro falando de qual tipo é esse campo no banco de dados [TIME, DATE OU TIMESTAMP]

	Exemplo

		Listando todos os funcinarios com mais de X anos de empresa e salario Y

		public List<Employee> funcinariosPorTempoESalario(double salario, Date inicio){

			TypedQuery<Employee> query = em.createQuery("SELECT e FROM Employee e WHERE e.startDate > :inicio AND e.salary > :salario", Employee.class);
			query.setParameter(":salario", salario);
			query.setParameter(":inicio", inicio, TemporalType.DATE);
			return query.getResultList();
		}


Executing Queries

	Query e TypedQuery tem 3 metodos para a execução das queries:

		* getResultList: retorna uma lista com os resultados da query. Caso nao seja encontrado nenhum valor correspondente a query, é retornado uma lista vazia.

		* getSingleResult: retorna apenas um valor para a qyuery exectutada. Caso nao seja encontrado nenhum valor, lança a exception NoResultException. Essa exceção nao causa rollback da transação.

		* executeUpdate: Usado para update ou delete. Retorna apenas o numero de linhas/entidades alteradas.



	Query e TypedQuery podem ser reutilizados enquanto o mesmo persiste context que foi usado para cria-lo ainda estiver ativo.


	Quando executar uma querie em uma transaction-scoped entityManager, onde os valores da query nao serão alterados, para evitar o overhead de adicionar esses valores, de forma desncessaria ao persistence context, use @TransactionAttribute para informar que esse metodo nao deve ser executado dentro de um transaction.

		@TransactionAttribute(TransactionAttributeType.NOT_SUPPORTED)
    	public List<Department> findAllDepartmentsDetached() {}


Special Result Types
	
	Sempre que no SELECT houver mais de um item, será retornado uma lista de de array de Object.

	public void displayProjectEmployees(String projectName) {
	    List result = em.createQuery(
	                            "SELECT e.name, e.department.name " +
	                            "FROM Project p JOIN p.employees e " +
	                            "WHERE p.name = ?1 " +
	                            "ORDER BY e.name")
	                    .setParameter(1, projectName)
	                    .getResultList();
	    int count = 0;
	    for (Iterator i = result.iterator(); i.hasNext();) {
	        Object[] values = (Object[]) i.next();
	        System.out.println(++count + ": " +
	                           values[0] + ", " + values[1]);
	    }
	}


	É possivel fazer algo bizarro: mapear esse retorno dentro de um objeto especifico, diretamente da query. Usando o operador NEW dentro do SELECT.
	A query acima retorna 2 campos: nome do funcionario e o nome do departamento que o mesmo trabalha.
	Sé criarmos uma classe que recebe esses 2 parametros no construtor, é possivel usar uma sintaxe bizarra para fazer esse mapemaneto:

	A classe [Pojo]:

		package example;

		public class EmpMenu {
		    
		    private String employeeName;
		    private String departmentName;

		    public EmpMenu(String employeeName, String departmentName) {
		        this.employeeName = employeeName;
		        this.departmentName = departmentName;
		    }

		    [...]
		}

		A query bizarra:

		public void displayProjectEmployees(String projectName) {
    		List<EmpMenu> result =
		    		em.createQuery("SELECT NEW example.EmpMenu(" +
		                                       "e.name, e.department.name) " +
		                       "FROM Project p JOIN p.employees e " +
		                       "WHERE p.name = ?1 " +
		                       "ORDER BY e.name",
		                       EmpMenu.class)
		           .setParameter(1, projectName)
		           .getResultList();
		    int count = 0;
		    for (EmpMenu menu : result) {
		        System.out.println(++count + ": " +
		                           menu.getEmployeeName() + ", " +
		                           menu.getDepartmentName());
		    }
		}



		Pontos a destacar:

		* No SELECT tem o operador NEW
		* O nome completo da classe
		* A classe tem que ter um construtor que receba o mesmo tanto de parametros, com os mesmos tipo e na mesma ordem.


Query Paging
	

	Metodos setFirstResult e setMaxResults
	Define qual vai ser a primeira linha retornada e a quantidade de registros a partir dessa linha
	Metodos disponiveis nas interfaces de Query


Queries and Uncommitted Changes
	
	As vezes alguma query é exeutada, é alterado algum valor nas entidades retornadas e, em seguida, antes da transação ser comitada, uma nova query pode ser executada, retornando o mesmo tipo de entidade que foi alterada, mas ainda esta em memoria.
	Nessa situação, pode ser que os valores alterados em memoria impliquem no resultado da query.

	Queries são executadas contra o banco de dados, não utilizam o persistence context.

	Nessa situação, o que o provider faz, por padrão é dar um flush no persiste context para que os resultados alterados em memoria sejam incorporados na query. MAS, isso tem um lado negativo: caso o resultado em memoria nao tenha nenhm efito sobre a query a ser executada, será feito um update/insert no banco de forma desncessaria, o que pode acarretar problemas de performance.

	Para isso, existe o metodo abaixo, que diz como o provider ira ser comportar nessa situação.

	EntityManager||Query.setFlushMode

		AUTO: Padrão. Antes da query, faz um flush do persistence context para que os resultados sejam incorporados na query.
		COMMIT: Diz para o entity manager que o resultado da query, nao tem nada a ver com os valores alterados em memoria. Manda brasa e executa essa query.

	Caso essa opção seja alterada no EntityManager, qualquer criada a patir dele, tera o valor definido. Caso a Query altere, esse valor tera precedencia.

	EntityManagers com escopo de transação, nao tem alterações persistidas [a cada nova transação, um novo sera criado com o valor padrao.


Query Timeouts

	Pode ser definido usando:

		Query.setHint("javax.persistence.query.timeout", TEMPO);


	Não totalmente portavel. Nem todo banco de dados e nem todo provider JPA tem obrigação de suportar.

	3 comportamentos podem acontecer:

		* Ser ignorado silenciosamente.
		* Ser aceito e lançar QueryTimeoutException. Deve ser tratada e não faz a transação rollback.
		* Ser aceito e lançar PersistenceException, que faz a transação rollback.


Bulk Update and Delete


	Query.executeUpdate: Usado para executar ações de UPDATE e DELETE em grande quantidade.
	Por que somente em grande quantidade:

		O EntityManager ja tem o metodo remove que serve para remover [DELETE] uma entidade por vez.
		Todas as mudanças feitas em uma entidade que esta no persitence context, quando o mesmo for comitado [fim da transação], já é atualizada no banco de dados.

		Ambas operações são feitas em uma quantidade reduzida de entidades por vez [1 no caso do DELETE, N no caso do UPDATE]


	Novamente: queries são executadas diretamente no banco de dados, não alterado persitence context. Uma entidade não sera detached do persistece context após ser removida do banco de dados dessa forma.

		Teste:
			Recuperar uma entidade
			Apagar a mesma através de Bulk Delete
			Alterar a entidade
			Comitar operação.


	O persitence detecta que uma JPQL UPDATE ou DELETE esta sendo executado e invalidar o cache, dessa forma, qualquer fetch operation que ocorra em seguida ira buscar os dados do banco de dados.

	CAUTION: Não use Native Query para executar essas operações, pois diferentemente do JPQL que trabalha com entidades e avisa quais entidades serao alteradas nessa operação [ao fazer o parse da query], esse mecanismo trabalha diretamente com tabelas. Então não é necessario o parse e o provider nao sabe quais tabelas estão sendo alteradas, deixando assim o persistence context em estado inconsistente.


	Se já esse problema trabalhar com Transaction-Scoped peristence context, com extended é ainda pior, pois, diferentemente do que acontece com os transaction-scoped que são atualizados apos um bulk operation, o mesmo nao acontece com esse tipo de persistence-context.


Bulk Delete and Relationships

	Da mesma forma que acontece ao utilizar EntityManager.remove, tambem é responsabilidade do desenvolvedor cuidar dos relacionamentos de uma entidade ao tentar remove-la

	Exemplo:

	------------  		-------------
	| Employee |  ---->	|Departament|
	------------  		-------------

	A query:

		DELETE FROM Department d WHERE d.name IN ('CA13', 'CA19', 'NY30') {}

	Vai dar ruim, pois Employee tem chave estrangeira [é dono do relacionamento] de Department. Se a query acima funcionasse, como ficaria os funcionarios pertencetesn aos departamentos removidos? Inconsistentes!

	Então, antes de remove esses departamentos, é necessario atualizar os departamentos que fazem referencia a eles:

	UPDATE Employee e 
	SET e.department = null
	WHERE e.department.name IN ('CA13', 'CA19', 'NY30') {}



Query Hints

	Ponto de extensão da API de queries.
	Possivel definir nas @NamedQuery, TypedQuery e Query
	Não é garatindo portabilidade
	Para evitar merdas, vendos são obrigados a ignorar hints desconhecidas (Não funciona, mas nao quebra)


Query Best Practices

	Sempre que possivel, use Named Queries
	Named Queries são parseadas ao iniciar da aplicação
	Named Queries forçam o uso de parametros na query, o que é uma otima pratica de segurança
	Queries que não terão seus valores alterados, é recomendado serem executadas fora de uma transação.
	Retorne somente o que for utilizar da entidade.
	Execute bulk operations em uma transação separada [preferencialmente], ou antes de qualquer operacao na transação corrente.




========================= CHAPTER 8 - Query Language ================================================================================================


Entidades nas queries são referidas por nome. Ou o atributo 'name' da anotação @Entity ou o nome simples da classe.
Queries são case-sentive APENAS para nome de entitdades e propriedades.


Select Queries

		SELECT e FROM Employee e

		Diferentemente do que a acontece em SQL, o alias da tabela é obrigado em JPQL
		DUVIDA: É obrigatorio para todas as entidades/tabela apenas para as listadas no FROM???


	Path Expressions

		O que é isso?

			é a navegação feita de uma entidade para seus campos ou outras entidades.

			SELECT e.name FROM Employee e; //state field path expression
			SELECT e.department FROM Employee e; //single-valued associatio
			SELECT e.projects FROM Employee e;// collection-valued association


		Não tem limites de navegação [e.department.manager.name...]
		É valido sempre da esquerda pra direita
		Nome pode continuar de um state field [e.name.id [X]] ou de uma collection-valued association [e.projects.name [X] Muitos projetos associados a um empregado]
		Não pode começar com uma embedable object. Sempre tem que começar com uma entidade.


	Entities and Objects

		É possivel [NÃO USE] usar a palavra chave OBJECT em uma query, apenas como um dica visual informando que o tipo retornado será um entidade:

			SELECT OBJECT(e){} FROM Employee e //DESCONSIDERE {}
	 
		Esse cenario tem uma limitacao de que single-valued association não podem ser retornadas usando essa palavra.

			SELECT OBJECT(e.department){} FROM Employee e //INVALIDO. //DESCONSIDERE {} 


		É possivel usar DISTINCT no SELECT para não retornar valores duplicados.

		No SELECT é possivel retornar:

			* entidades
			* single-valued association de entidades. Que no final das contas tambem são entidades //SELECT e.department
			* state field. //SELECT e.name
			* Embedable objects.


		Esses, diferentemente do que acontece com todas as entidades listadas acima, não terão seus estados gerenciados [SOMENTE ENTIDADES SÃO GERENCIADAS]. O que isso significa:

			Se voce retonar um employee e altera o contactInfo do mesmo [contactInfo sendo um objeto embedado], essas alterações serão persistitidas.
			Se voce retorna somente o contactInfo do employee [SELECT e.contactInfo ...] e alterar esse objecto, nada sera persistindo.


	Combining Expressions

		
		É possivel retornar mais de um valor por SELECT [oh really?]. Por isso, quer se dizer que pode ser retornado apenas um ou algums campos de uma entidade.

		DUVIDA: Pode ser retornado mais de uma entidade??? SELECT e, d FROM Employee e, Department d ...


		Um exemplo:

			SELECT e.name, e.salary FROM Employee e

		Nessa query serão retornados apenas o nome[uma String] e o salario [um Double] de cada funcionario.
		Em java, chegara uma lista [getResultList] ou apenas um [getSingleResult] de array de object, com duas posições. Cada uma referente a um campo retornado na query.


	Constructor Expressions

		Para a query acima, em vez de retornar um array de object e fazer o parse dos objetos na mão, é possivel fazer uso do constructor expression:

		SELECT NEW example.EmployeeDetails(e.name, e.salary) {} FROM Employee e; // {} NAO FAZ PARTE

		O que isso faz:

			* Para cada resultado, instancia a classe EmployeeDetails passando os valores retornados para seu construtor.
			* Retorna uma lista de EmployeeDetails em vez de um array de objectos.
			* A classe instanciada obrigatoriamente precisa ter um construtor equivalente aos argumentos informados.
			* A classe nao precisar ter seu estado no banco de dados = não precisa ser uma entidade.

		DUVIDAS: 
			O retornado será managed pelo EntityManager?
			Caso seja uma entidade [pode ser?] e seja gerenciado, os resultados serão persistidos? [Nesse cenario, sim. Acho]



FROM Clause
	

	Identification Variables

		Tem que seguir as regras java de nomes para variaveis.
		Range variable: "alias" utilizando para representar uma entidade. [Range porque aborda toda a entidade]


	Joins

		Joins acontecem quando:

			Two or more range variable declarations are listed in the FROM clause and appear in the select clause.

				SELECT p FROM Employee e, Phone p
					WHERE e.department = "RH";

			The JOIN operator is used to extend an identification variable using a path expression.
				

			A path expression anywhere in the query navigates across an association field, to the same or a different entity.

				SELECT e.department FROM Employee e
					
			One or more WHERE conditions compare attributes of different identification variables:

				SELECT e FROM Employee e
					WHERE e.department = "RH";


		Produtos catersianos das tabelas são gerados quando duas range variables são expressas no FROM, mas sem nada no WHERE:

			SELECT p FROM Employee e, Phone p;

		Cada linha de Employee será pareada com cada linha de Phone. Se Employee e Phone tem 2 linhas cada, serão retornados 4 valores.


JOIN Operator and Collection Association Fields

	JOIN = INNER JOIN

	Exemplo:

		SELECT p FROM Employee e JOIN e.phones p

		vira em SQL

		SELECT p.id, p.phone_num, p.type, p.emp_id
		FROM emp e, phone p
		WHERE e.id = p.emp_id

	DUVIDA: Isso ainda é igual e funciona:

		SELECT p  FROM Employee e, Phone p
			where e.phone = p;

			OU

		SELECT e.phones FROM Employee e
	RESPOSTA: SIM!


	Essa query faz o JOIN entre Employee e seus Phones. Traz todos os telefones associados com algum funcionario da empresa.
	Apesar do tipo Phone não aparecer explicitamente nessa query, como o relacionamento "e.phones" é desse tipo, o Phone é inferido.
	A associação entre Employee e Phone esta toda definida no mapeamento do relacionamento.
	Apesar de na classe Employee ter uma List/Collection de Phone a range variable esta se referindo a entidade e não a lista. Assim, caso fosse mais prudente, poderia ser retonado apenas os numeros dos telefones:

	SELECT p.number FROM Employee e JOIN e.phones p

	Como dito antes, uma path expression nao pode continuar de uma collection-valued association. MAS, nesse exemplo, como foi feito o JOIN e agora essa association é a raiz de uma expression, pode. Isso ainda nao pode:

	SELECT e.phones.number FROM Employee e




JOIN Operator and Single-Valued Association Fields


	SELECT d FROM Employee e JOIN e.department d

		OU

	SELECT e.department FROM Employee e

		OU ???

	SELECT d FROM Employee e, Department d
		WHERE e.department = d;


	Path expression = INNER JOIN = JOIN.
	Fique esperto com isso.


	Joins logicos, feitos entre as entidades, podem ser diferentes do que os joins fisicos, feito entre as tabelas.
	Exemplo:

	JPQL:

		Faz JOIN entre 4 entidades

		SELECT DISTINCT e.department
			FROM Project p JOIN p.employees e
			WHERE p.name = 'Release1' AND
			      e.address.state = 'CA'

	SQL:

		Faz JOIN entre 5 entidades, ja que relacionamento entre Project e Employee é ManyToMany.
		Existe employee_project no meio, fazendo o mapeamento:


		SELECT DISTINCT d.id, d.name
		FROM project p, emp_projects ep, emp e, dept d, address a
		WHERE p.id = ep.project_id AND
		     ep.emp_id = e.id AND
		      e.dept_id = d.id AND
		      e.address_id = a.id AND
		      p.name = 'Release1' AND
		      a.state = 'CA'


Multiple Joins

	
	JOINS cascateados podem ser executados sempre que necessario.
	A query abaixo retorna todos os projetos dos funcionarios que estão em algum departamento:

	SELECT DISTINCT p
	FROM Department d JOIN d.employees e JOIN e.projects p


Map Joins
	
	Considerando o mapeamento abaixo:

	public class Employee {
	    private Map<String, String> phones;
	}

	Sendo a chave do map o tipo de telefone (residencial, comercial, etc) e o value, o telefone mesmo.

	Ao executar:

		SELECT p FROM Employee e JOIN e.phones p;

	O comportamento padrão será retornar o value desse map.
	Para explicitar esse comportamento fazendo:
	{
		SELECT VALUE(p) FROM Employee e JOIN e.phones.
	}

	Caso seja desejado retonar a chave do map, use a keyword KEY:

	{
		SELECT VALUE(p), KEY(P) FROM Employee e JOIN e.phones;

		ou

		SELECT VALUE(p) FROM Employee e JOIN e.phones;
		WHERE KEY(p) IN ('Trabalho', 'Celular')
	}

	Existe ainda a opção de retornar uma Entry, contendo a chave e o valor:

	{
		SELECT ENTRY(p) FROM Employee e FROM e.phones;
	}

	VALUE e KEY podem ser usandos no WHERE, diferentemente do ENTRY que esta restrito ao SELECT.


Outer Joins

	O que é?

	Em um INNER JOIN entre Employee e Department, só serão retornados os Employees pertencentes a algum departamento e seus respectivos departamentos.
		
		SELECT e, d
		FROM Employee e JOIN e.department d


	No LEFT [OUTER] JOIN serão retornados todos os Employees e seus respectivos departamentos, mas tambem todos os outros funcionarios sem departamentos.

		SELECT e, d
		FROM Employee e LEFT JOIN e.department d

	Em SQL, um OUTER JOIN é representado usando ON:

	{
		SELECT e.id, e.name, e.salary, e.manager_id, e.dept_id, e.address_id, d.id, d.name
		FROM employee e LEFT OUTER JOIN department d
		ON (d.id = e.department_id)
	}

	No JPA 2.1 é possivel usar ON adicionais para fazer comparações. EM OUTER JOIN, se for usando WHERE, o mesmo será considerado como INNER JOIN. 


Fetch Joins

	
	Por que usar?

		Mapeamentos @xptoToMany por padrão são lazy loading, ou seja, ao retornar um Employee, sua lista de telefone não retornada. Quando a aplicação tentar acessa-la, sera feito um novo hit no banco de dados para trazer essa lista.
		Caso seja sabido que essa lista ja sera usada, pode ser usada o FETCH JOIN para evitar essa segunda ida ao banco.


	Semantica:

		SELECT e FROM Employee e FETCH JOIN e.phones

		É diferente por não ter um identification variable para os telefones.
		A lista não parte do resultado da query, por isso a ausencia da variavel.
		A lista pode ser acessada de forma segura depois que o objeto for detached. Nada de View Transaction \o/


		
WHERE Clause

	Input Parameters

		Se dividem em:

			* Positional notation:

				Definido usando ponto de esclamação e o numero
				SELECT  e FROM Employee e WHERE e.name = "?1"

			* Named identification:

				Defindo usando 2 pontos e uma identificação:
				SELECT e FROM Employee e WHERE e.name = ":name"


		Pontos importante:

			Caso o parametro ocorra em mais de algum lugar na query, ao defini-lo uma vez através da interface Query {},o valor ja será substitido em todos os lugares;



	BETWEEN Expressions


		Usando para definir um range no qual o valor deve estar contido. BETWEEN usa >= e <=.
		Exemplo:

		SELECT e FROM Employee e WHERE e.salary BETWEEN 40000 AND 45000

		Tambem aceita o NOT


	LIKE Expressions

		Igual o do SQL.

		Caso no meio do pattern utilizado o _ ou % deva ser um ligeral, use ESCAPE:

		SELECT d FROM Department d WHERE d.name LIKE 'QA\_%' ESCAPE '\''; //QA_East


	Subqueries

		Pode ser usado no WHERE ou no HAVING
		Pense uma subquerie como uma querie sendo executada para cada 'linha' da main querie.
		Identificadores definidos na query pai podem ser acessados na querie filha.
		Se uma subquerie tive outras subqueries, a regra acima se aplica.
		Uma subquerie pode sobrescrer o identificador definido no pai. Comportamento nao completamente portavel.

		Retorna o maior salario:
		SELECT e
		FROM Employee e
		WHERE e.salary = (SELECT MAX(emp.salary)
		                  FROM Employee emp)


		Valido, faz o mesmo que a querie acima, mas impede que subquerie acesse a variavel do pai e não é totalmente portavel.
		SELECT e
		FROM Employee e
		WHERE e.salary = (SELECT MAX(e.salary)
		                  FROM Employee e)


		Exemplo de um subquerie acessando o identificador da main query:

			Retorna todos os funcionarios com celular.
			O operador EXISTS valida se uma lista não é vazia.
			SELECT e
			FROM Employee e
			WHERE EXISTS (SELECT 1
			              FROM Phone p
			              WHERE p.employee = e AND p.type = 'Cell')


			Essa querie tambem pode ser escrita assim (melhor):
			SELECT e
			FROM Employee e
			WHERE EXISTS (SELECT 1
			              FROM e.phones p
			              WHERE p.type = 'Cell')


	IN Expressions

		Igual ao do SQL.
		Resultado tem que estar contido na lista.
		Aceita o operador NOT, para negar a lista.


	Collection Expressions

		IS [NOT] EMPTY

			Usado para collection association [@xptoToMany]
			Utilizado para ver se a coleção tem pelo menos um valor. Em Java, se a lista não esta vazia.
			Ou a negação, para ver se a lista esta vazia.

			Retorna todos os funcionarios que tem algum contato:
			Util para demissões em massa.
			SELECT e FROM Employee WHERE e.phones IS NOT EMPTY

			É traduzido para uma subquerie.
			Exemplo:

			Todo funcinoario que tem alguem para quem ele da instruções é um gerente. Para achar gerentes:

			SELECT e FROM Employee e WHERE e.directs IS NOT EMPTY

			É equivalente a:

			SELECT manager FROM Employee manager WHERE (
				SELECT COUNT(e) FROM Employee e WHERE e.manager = manager) > 0

		MEMBER OF 

			Utilizado para ver se um elemento faz parte de uma lista

			Retorna todos os funcionarios que estão no projeto XPTO:
			SELECT e
			FROM Employee e
			WHERE :project MEMBER OF e.projects



	EXISTS Expressions
	
		Usado para validar se uma Subqueries retornou algum valor.

		A query abaixo retorna todos os funcionarios que tem celular:
		SELECT e
		FROM Employee e
		WHERE NOT EXISTS (SELECT p
		                  FROM e.phones p
		                  WHERE p.type = 'Cell')


		Traz todos os funcionarios que não estao no projeto XTPO:
		SELECT e FROM Employee e
			WHERE NOT EXISTS (
				SELECT p FROM e.projects
				WHERE p.name = ":XTPO")


	ANY, ALL, and SOME Expressions


		Usado para fazer comparação utilizando o resultado de uma subquerie.

		A querie abaixo retorna todos os gerentes que recebem menos do que todos os seus gerenciados:
		Para essa condição ser verdadeira, TODOS os gerenciados tem que ganhar mais do que o gerente.

		SELECT e
		FROM Employee e
		WHERE e.directs IS NOT EMPTY AND
		      e.salary < ALL (SELECT d.salary
		                      FROM e.directs d)



		Ja utilizando ANY ou SOME, que são equivalentes, essa condição seria verdadeira caso o gerente tivesse algum funcionario que ganhasse mais do que ele:

		SELECT e
		FROM Employee e
		WHERE e.directs IS NOT EMPTY AND
		      e.salary < ANY (SELECT d.salary
	                      FROM e.directs d)



	Inheritance and Polymorphism


		Para fim de exemplo, considere as classes DesignProject e QualityProject que herdam de Project.

		Ao executar a query abaixo, serão retornados todos os projetos que tem algum funcionario, indiferente do tipo do mesmo:

		SELECT p FROM Project p WHERE p.employees IS NOT EMPTY

		Caso quisesse retornar somente os DesignProject, poderia fazer:

		SELECT p FROM DesignProject p WHERE p.employees IS NOT EMPTY

		É possivel usar o  operador Type para fazer uma comparação de tipo na clausula WHERE em vez da SELECT:
		Type recebe um parametro e será retornado o tipo do mesmo. Esse valor retronado nao sera uma String.
		Esse valor pode ser usado para fins de comparação.

		Exemplo:

		SELECT p
		FROM Project p
		WHERE TYPE(p) = DesignProject OR TYPE(p) = QualityProject

		Perceba que os tipos comparados nao estro entre ''.
		O tipo pode ser recebido como parametro:

		SELECT p
		FROM Project p
		WHERE TYPE(p) = :projectType


	Downcasting [JPA 2.1]

		TREAT

			Para acessar valores especificos de uma subclass, use o operador TREAT:

			SELECT p
			FROM Project p
			WHERE TREAT(p AS QualityProject).qaRating > 4
			       OR  TYPE(p) = DesignProject

			Semantica:
				TREAT(p-> path param do supertipo AS Subclass).variavelDefinaNoFilho


		Treat no JOIN

			Quando é feito um JOIN, normalmente são inclusos todos os subtipos. Para espeficiar isso, use:

			SELECT e, q.name, q.qaRating
			FROM Employee e JOIN TREAT(e.projects AS QualityProject) q
			WHERE q.qaRating > 4

			Retorna todos os funcionarios que trabalharam em um projeto de quailidade com avaliação maior que 4



Scalar Expressions

	Literals

		São suportados strings, numerics, booleans, enums, entity types, e temporal types.
		Booleans são representados por TRUE ou FALSE;

		Para representar Enuns, use o nome qualificado da classe:

		SELECT e
		FROM Employee e JOIN e.phoneNumbers p
		WHERE KEY(p) = com.acme.PhoneType.Home;

		Irá retornar todos os funcionarios que tem um telefone residencial.


		Tipos temporais saão definidos usando a seguinte forma.

		{d 'yyyy-mm-dd'}               e.g. {d '2009-11-05'}
		{t 'hh-mm-ss'}                 e.g. {t '12-45-52'}
		{ts 'yyyy-mm-dd hh-mm-ss.f'}   e.g. {ts '2009-11-05 12-45-52.325'}

		Sendo:
			* t|d|ts = caracter informando qual o tipo temporal.
			* Um espaço
			* O valor do literal.



	Function Expressions

		Uma porrada de funções que podem ser usadas.

		Sobre SIZE:

			Retorna o tamanho de uma lista.
			Atalho para subquerie;

			Retorna todos os departamentos com 2 funcionarios.
			{
				SELECT d
				FROM Department d
				WHERE SIZE(d.employees) = 2
			}
				=

			{
				SELECT d
				FROM Department d
				WHERE (SELECT COUNT(e)
				       FROM d.employees e) = 2
			}

	Native Database Functions [JPA 2.1]


		Pode chamar funções proprias do banco de dados, sejam essas definidas pelo banco ou pelo adm.
		Use FUNCTION
		O resultado da função tem que ser um literal e pode ser usada em qualquer lugar onde o mesmo tipo de literal seria.

		Semantica:

			FUNCION('nomeDaFuncap','parametroDeEntrada1', 'parametroDeEntradaN')

		Exemplo:

			SELECT DISTINCT e
			FROM Employee e JOIN e.projects p
			WHERE FUNCTION('shouldGetBonus', e.department.id, p.id)

			Espera-se que essa função retone um boolean sinalizando se o funcionario deve receber um bonus. (o/)	



CASE Expressions

	Primeira forma:

		A expressão é validada em cada passagem. Mais ou menos como uma clausula do IF para cada expressão.

		SELECT p.name,
       	CASE 
       		WHEN TYPE(p) = DesignProject THEN 'Development'
            WHEN TYPE(p) = QualityProject THEN 'QA'
            ELSE 'Non-Development'
       	END
		FROM Project p
		WHERE p.employees IS NOT EMPTY


	Segunda forma:

		A expressão é validada somente uma vez. Mais ou menos como somente uma clausula do IF para geral.

		SELECT p.name,
		    CASE TYPE(p)
	            WHEN DesignProject THEN 'Development'
	            WHEN QualityProject THEN 'QA'
	            ELSE 'Non-Development'
	       END
		FROM Project p
		WHERE p.employees IS NOT EMPTY

	Terceira forma:

		As expressoes são resolvidas em ordem e a primeira a retornar um valo não nulo, fica sendo o resultado da expressão.

		Retorna o nome OU o id do departamento
		SELECT COALESCE(d.name, d.id) FROM Department d;

	Quarta forma:

		Resolve e compara as duas formas. Caso os valores sejam iguais, retorna null. Caso não, retorna o valor da primeira.
		Util para excluir valores. A query abaixo retorna a contagem de todos os departamentos e de todos que não se chamam "QA":

			   //TODOS,  //Todos que não se chamam QA.
		SELECT COUNT(*), COUNT(NULLIF(d.name, 'QA'))FROM Department d




ORDER BY Clause

	Quase totalmente igual ao SQL.

	Isso não é permitido:

		SELECT e.name FROM Employee e ORDER BY e.salary DESC 
		ORDER BY esta limitado ao mesmo state field do SELECT.

	Mas isso é:

		SELECT e FROM Employee e ORDER BY e.name DESC

	* O padrão é ordenção ascendente.
	* É possivel definir e usar variaveis no ORDER BY:

		SELECT e.name, e.salary * 0.05 AS bonus, d.name AS deptName
		FROM Employee e JOIN e.department d
		ORDER BY deptName, bonus DESC

		variaveis deptName e bonus foram definidas no SELECT.


Aggregate Queries

	
	Exemplo:

		Retorna a media salarial[1] de todos os departamentos[3] com media acima de 50000[4], excluindo dessa o salario dos gerentes[2]  

		SELECT d.name, AVG(e.salary) // 1
		FROM Department d JOIN d.employees e
		WHERE e.directs IS EMPTY //2
		GROUP BY d.name //3
		HAVING AVG(e.salary) > 50000 //4


	Aggregate Functions


		São 5:

			 * AVG: Aceita um state field, que tem que ser numerico e retorna um Double com a media dos valores.
			 * COUNT: Aceita um state field ou um single-valued association e retorna a contagem de valores. Pode ser usada junto com DISTINCT para eliminar valores duplicados; 

				SELECT COUNT(e.firstName), COUNT(DISTINCT e.firstName) FROM Employee e;

			* MAX e MIN: Retornam os maiores e menores valores do state field passado

			* SUM: Soma os valores de um state field informado. Retoro é igual ao tipo numerico informado (se state field Double, retorna Double.)



GROUP BY Clause

	* Igual o SQL.
	* Aceita somente single-valued expression
	* Define como o resultado será agregado.
	* O mesmo valor usado no GROUP BY DEVE ser usado em todos outras expressoes não agregadoras:

		Nesse exemplo, a mesma expressão defininda no GROUP BY esta no SELECT

		SELECT d.name, COUNT(e), AVG(e.salary)
		FROM Department d JOIN d.employees e
		GROUP BY d.name


HAVING Clause

	* Como um segundo WHERE, mas executado sobre os valores retornados no GROUP BY.
	* Ordem: WHERE (Filtra) -> GROUP BY (Agrupa) -> HAVING (Filtra)
	* Sua clausula logica esta limitada aos valores usados no GROUP BY.
	* Pode usar funções agregadoras sobre os valores do GROUP BY ou usar funções agregadoras definidas no SELECT.

	Exemplo:

		SELECT e, COUNT(p)
		FROM Employee e JOIN e.projects p
		WHERE XYZ
		GROUP BY e
		HAVING COUNT(p) >= 2



Update Queries


	* Igual ao SQL.
	* Novamente, o WHERE pode executar subqueries que usam variaveis definidas no nivel acima, nesse caso, o UPDATE:

		Exemplo:

		UPDATE Employee e
		SET e.salary = e.salary + 5000
		WHERE EXISTS (SELECT p
		              FROM e.projects p
		              WHERE p.name = 'Release2')


Delete Queries 

	* Igual ao SQL.
	* DELETE caga para o CASCADES RULES. Ou seja, mesmo que a entidade tenha definido nos seus relacionamentos DELETE nas regras do cascade, isso nao se aplica aqui (apenas para o EntityManager.remove)






========================= CHAPTER 9 - Criteria API ====================================================================================================

The Criteria API

	
	JPQL

	SELECT e
	FROM Employee e
	WHERE e.name = 'John Smith'

	O equivalente em Criteria

	CriteriaBuilder cb = em.getCriteriaBuilder();
	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp).where(cb.equal(emp.get("name"), "John Smith"));

	CriteriaQuery é como se fosse a String da Query.
	CriteriaQuery sempre gera TypedQuery.
	CriteriaBuilder criar a CriteriaQuery e tambem as condições [WHERE, etc] da query;

	CriteriaBuilder:
		
		* createQuery(Class<T>): retorna um CriteriaQuery tipado, que será o resultado da query.
		* createQuery(): retorna um CriteriaQuery sem tipo, no qual o retorno da query sera Object.
		* createTupleQuery(): Usado para queries de relatorio, onde o SELECT contem mais de um tipo. Equivalente a createQuery(Tuple.class). Util para quando o tipo de retorno é mais de um objecto. Sendo eles entidades, campos ou a combinação de ambos.


	Comparação JPQL e Criteria API

	JPQL Clause	 	Criteria API Interface		Method

	SELECT	 		CriteriaQuery				select()
					Subquery					select()
	FROM	 		AbstractQuery				from()
	WHERE	 		AbstractQuery				where()
	ORDER BY	 	CriteriaQuery				orderBy()
	GROUP BY	 	AbstractQuery				groupBy()
	HAVING	 		AbstractQuery				having()

Criteria Objects and Mutability

	CriteriaBuilder e seus objetos imutaveis

		A maioria dos metodos invacados em CriteriaBuilder cria objetos que são imutaveis. CriteriaQuery e  Subquery são os unicos mutaveis. Isso significa que ao chamar um metodo que recebe/cria algo imutavel, todas as informações necessarias para o pleno uso do mesmo, devem ser informados no momento da criação.

	CriteriaQuery.select

		Quando chamado 2 ou mais vezes, a ultima é que fica. Esse metodo define o que sera retornado.

	CriteriaQuery.from

		Quando chamado 2 ou mais vezes, vai adicionado os resultaos. Esse metodo define de qual entidade/tabela os valores serão recuperados;



Query Roots and Path Expressions

	
	Query Roots
		
		Criadas a partir da chamada do CriteriaQuery.from;
		Equivale a uma variavel definida seja no FROM [ ...FROM Department d] ou atráves de JOIN [JOIN e.deparment d]
		Chamadas ao metodo from são cumulativas;

			SELECT DISTINCT d
			FROM Department d, Employee e
			WHERE d = e.department

						=

			CriteriaQuery<Department> c = cb.createQuery(Department.class);
			Root<Department> dept = c.from(Department.class);
			Root<Employee> emp = c.from(Employee.class);
			c.select(dept)
			 .distinct(true)
			 .where(cb.equal(dept, emp.get("department")));


	Path Expressions

							--------------  		-------------			
							| Path [get] |  ---->	|Root 		|    ---->
							--------------  		-------------


		SELECT e
		FROM Employee e
		WHERE e.address.city = 'New York'

		=

		CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
		Root<Employee> emp = c.from(Employee.class);
		c.select(emp)
		 .where(cb.equal(emp.get("address").get("city"), "New York"));

		 O metodo get, que vem de Path, é equivalente ao uso do [.] no JPQL.
		 Como no JQPL e SQL, a tabela utilizada no FROM, nesse caso o Root da query, é o ponto de inicio da navegação [o que vai ser retornado, JOINs e etc.]
		 O Root deve ser guardado em uma variavel. Multiplas invocações a esse metodo pode gerar produto cartesiano da tabela:

		 	Root<Employee> emp = c.from(Employee.class);
		 	c.select(emp);
		 	... etc
		 	Root<Employee> emp2 = c.from(Employee.class);

		 	SELECT emp FROM Employee emp, Employee emp2 [...]



The SELECT Clause
	
	Selecting Single Expressions

		CriteriaQuery<T> select(Selection<? extends T> selection);

		Deve ser compativel com o tipo definido ao criar a CriteriaQuery:

		CriteriaQuery<Employee> query = cb.createQuery(Employee.class);
		Root<Employee> employee = query.from(Employee.class);
		quey.select(employee);
		//Root, usado no Select, é do mesmo tipo informado no createQuery do CriteriaBuilder.

		É possivel retornar uma entidade inteira ou apenas um campo da mesma.


	Selecting Multiple Expressions


		Se o resultado for Tuple, entao CompoundSelection<Tuple> deve ser passado no select.
		Se o resultado for um objeto que será criado, entao deve ser informado CompoundSelection<[T]> , sendo T o tipo do objeto
		Se o resultado for um array de Objetos, então CompoundSelection<Object[]> deve ser usado.

		//DUVIDA: Coloque exemplos
		

The FROM Clause

	
	Inner and Outer Joins


		Criados usando o metodo join em Root.
		O mais usando recebe uma String, como o nome do parametro que será feito o JOIN e, opicionalmente pode receber atributo JoinType que especifica qual sera o tipo do join. Se omitido, o padrão é INNER JOIN
		A interface_ retornada tem como generics o tipo de origem e o destino do join.

		Exemplo:

			SELECT e FROM Employee e JOIN e.department d;

				=

			....
			Root<Employee> employee = query.from(Employee.class)
			Join<Employee, Department> department = employee.join("department")

		A chamada a join pode ser encadeada:

			Join<Employee,Project> project = dept.join("employees").join("projects");


		Como sempre, qualquer coisa que envolve maps é uma desgraça separada.
		Para fazer join em maps, deve ser usado o metodo joinMap que retorna um interface_ com 3 generics:

		Exemplo:

			SELECT e.name, KEY(p), VALUE(p) FROM Employee e JOIN e.phones p

				= 

			Root<Employee> employee = query.from(Employee.class)
			MapJoin<Employee,String,Phone> phone = employee.joinMap("phone");
			query.multiSelect(employee.get("name"), phone.key(), phone.value());

			Sendo:
				1º Tipo da origem do join (Employee)
				2º Tipo da chave do map (String representado o tipo do telefone (casa, trabalho, etc))
				3º Tipo da valor do map (Phone)


	Fetch Joins


		Retorna a lista que seria carregada de modo lazy;
		Use o metodo fetch no root passando o path da association;

		SELECT e FROM Employee e JOIN FETCH e.address

			=

		CriteriaQuery<Employee> query = cb.createQuery(Employee.class);
		Root<Employee> employee = query.from(Employee.class);
		employee.fetch("address");
		query.select(employee);


The WHERE Clause

	
	Cada nova chamada para CriteriaQuery.where ira fazer com que as expressões anteriores sejam descartadas.


	Building Expressions

		JP QL to CriteriaBuilder Predicate Mapping
		------------------------------------------

		JP QL Operator		CriteriaBuilder Method
		AND					and()
		OR					or()
		NOT					not()
		=					equal()
		<>					notEqual()
		>					greaterThan(), gt()
		>=					greaterThanOrEqualTo(), ge()
		<					lessThan(), lt()
		<=					lessThanOrEqualTo(), le()
		BETWEEN				between()
		IS NULL				isNull()
		IS NOT NULL			isNotNull()
		EXISTS				exists()
		NOT EXISTS			not(exists())
		BETWEEN				between()
		IS NULL				isNull()
		IS NOT NULL			isNotNull()
		EXISTS				exists()
		NOT EXISTS			not(exists())
		IS EMPTY			isEmpty()
		IS NOT EMPTY		isNotEmpty()
		MEMBER OF			isMember()
		NOT MEMBER OF		isNotMember()
		LIKE				like()
		NOT LIKE			notLike()
		IN					in()
		NOT IN				not(in())

		--------------------------------------------------
		JP QL to CriteriaBuilder Scalar Expression Mapping
		--------------------------------------------------

		JP QL Expression	CriteriaBuilder Method
		ALL					all()
		ANY					any()
		SOME				some()
		-					neg(), diff()
		+					sum()
		*					prod()
		/					quot()
		COALESCE			coalesce()
		NULLIF				nullif()
		CASE				selectCase()

		------------------------------------------
		JP QL to CriteriaBuilder Function Mapping
		------------------------------------------

		JP QL Function		CriteriaBuilder Method
		ABS					abs()
		CONCAT				concat()
		CURRENT_DATE		currentDate()
		CURRENT_TIME		currentTime()
		CURRENT_TIMESTAMP	currentTimestamp()
		LENGTH				length()
		LOCATE				locate()
		LOWER				lower()
		MOD					mod()
		SIZE				size()
		SQRT				sqrt()
		SUBSTRING			substring()
		UPPER				upper()
		TRIM				trim()

		---------------------------------------------------
		JP QL to CriteriaBuilder Aggregate Function Mapping
		---------------------------------------------------

		JP QL Aggregate Function	CriteriaBuilder Method
		AVG							avg()
		SUM							sum(), sumAsLong(), sumAsDouble()
		MIN							min(), least()
		MAX							max(), greatest()
		COUNT						count()
		COUNT DISTINCT				countDistinct()



	Predicates

		Multiplos predicates funcionam como AND
		CriteriaBuilder.gt é especifico  para numeros enquanto CriteriaBuilder.greaterThan serve para qualquer parametro.

	Literals

		Muitos metodos são sobrecarregados para receber tanto literais e Expressions. Maaas, em alguns casos, apenas Expressions sao aceitos. Para esses casos use:

			CriteriaBuilder.literal(Valor) = Ira retornar uma expressao representando o valor.
			CriteriaBuilder.nullLigeral(Tipo.class);



	Parameters

		Diferentemente do que acontece com JQPL onde um pametro pode ser definido usando o nome precedido por :, em CriteriaAPI, nao é tao simples.
		Deve ser criado um ParameterExpression, que pode ser nomeado ou não, para que ao definir a query [EntityManager.createQuery], o mesmo seja substituido;


		SELECT emp FROM Employee emp WHERE emp.depto.name = :deptoName

			VIRA:

		CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
		Root<Employee> emp = c.from(Employee.class);
		c.select(emp);
		ParameterExpression<String> deptName =
		    cb.parameter(String.class, "deptName");
		c.where(cb.equal(emp.get("dept").get("name"), deptName));



	Subqueries

		O metodo CriteriaQuery.subquery:

			* Recebe como argumento o tipo que deve ser retornado.
			* É uma query comum, como a definida atraves do metodo CriteriaBuilder.createQuery

		O metodo Subquery.correlate faz o mesmo que from, porem, ao inves de receber uma tipo de uma interface_, recebe ou um Root ou um Join.
		Exemplo 3. 


		==========================================================================================================

		(1)

	        CriteriaBuilder cb = em.getCriteriaBuilder();
	        CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	        Root<Employee> emp = c.from(Employee.class);
	        c.select(emp);

	        if (projectName != null) {
	            Subquery<Employee> sq = c.subquery(Employee.class);
	            Root<Project> project = sq.from(Project.class);
	            Join<Project,Employee> sqEmp = project.join("employees");
	            sq.select(sqEmp)
	              .where(
	              	cb.equal(
	              		project.get("name"), cb.parameter(String.class, "project")
	              		)
	              	);
	            
	            criteria.add(cb.in(emp).value(sq));
	        }

		SELECT emp 
			FROM Employee emp
			WHERE emp IN (SELECT sqEmp 
							FROM Project project 
							JOIN project.employees sqEmp
							WHERE project.name = :project)




	==========================================

	(2)

	CriteriaBuilder cb = em.getCriteriaBuilder();
	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp);

	if (projectName != null) {
	    Subquery<Project> sq = c.subquery(Project.class);
	    Root<Project> project = sq.from(Project.class);
	    Join<Project,Employee> sqEmp = project.join("employees");
	    sq.select(project)
	      .where(cb.equal(sqEmp, emp),
	             cb.equal(project.get("name"),
	                      cb.parameter(String.class,"project")));
	    criteria.add(cb.exists(sq));
	}


	SELECT e
	FROM Employee e
	WHERE EXISTS (SELECT p
	              FROM Project p JOIN p.employees emp
	              WHERE emp = e AND
	                    p.name = :name)

	===========================================

	(3)

	CriteriaBuilder cb = em.getCriteriaBuilder();
	CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
	Root<Employee> emp = c.from(Employee.class);
	c.select(emp);


	if (projectName != null) {
	    Subquery<Project> sq = c.subquery(Project.class);
	    Root<Employee> sqEmp = sq.correlate(emp);
	    Join<Employee,Project> project = sqEmp.join("projects");
	    sq.select(project)
	      .where(cb.equal(project.get("name"),
	                      cb.parameter(String.class,"project")));
	    criteria.add(cb.exists(sq));
	}

	SELECT e
	FROM Employee e
	WHERE EXISTS (SELECT p
	              FROM e.projects p
	              WHERE p.name = :name)

	=================================================

	(4)

	CriteriaQuery<Project> c = cb.createQuery(Project.class);
	Root<Project> project = c.from(Project.class);
	Join<Project,Employee> emp = project.join("employees");
	Subquery<Number> sq = c.subquery(Number.class);
	Join<Project,Employee> sqEmp = sq.correlate(emp);
	Join<Employee,Employee> directs = sqEmp.join("directs");
	c.select(project)
	 .where(cb.equal(project.type(), DesignProject.class),
	        cb.isNotEmpty(emp.<Collection>get("directs")),
	        cb.ge(sq.select(cb.avg(directs.get("salary"))),
	                  cb.parameter(Number.class, "value")));



	SELECT p
	FROM Project p JOIN p.employees e
	WHERE TYPE(p) = DesignProject AND
	      e.directs IS NOT EMPTY AND
	      (SELECT AVG(d.salary)
	       FROM e.directs d) >= :value




	IN Expressions

		Aceita apenas um argumento

		SELECT e
		FROM Employee e
		WHERE e.address.state IN ('NY', 'CA')

		CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
		Root<Employee> emp = c.from(Employee.class);
		c.select(emp)
		 .where(cb.in(emp.get("address")
		                 .get("state")).value("NY").value("CA"));


		 OU

		CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
		Root<Employee> emp = c.from(Employee.class);
		c.select(emp)
		 .where(emp.get("address")
		           .get("state").in("NY","CA"));


	The ORDER BY Clause

		Equivalente ao metodo CriteriaQuery.orderBy
		Recebe uma lista de Order criadas atraves dos metodos CriteriaBuilder.asc e CriteriaBuilder.desc, que organizam em ordem ascendente e descedente, respectivamente.


		SELECT d.name, e.name
		FROM Employee e JOIN e.dept d
		ORDER BY d.name DESC, e.name

		=

		CriteriaQuery<Tuple> c = cb.createQuery(Tuple.class);
		Root<Employee> emp = c.from(Employee.class);
		Join<Employee,Department> dept = emp.join("dept");
		c.multiselect(dept.get("name"), emp.get("name"));
		c.orderBy(cb.desc(dept.get("name")),
		          cb.asc(emp.get("name")));


	The GROUP BY and HAVING Clauses

		Equivalente ao metodo CriteriaQuery.groupBy e CriteriaQuery.having

		SELECT e, COUNT(p)
	    FROM Employee e JOIN e.projects p
	    GROUP BY e
	    HAVING COUNT(p) >= 2
	    
		CriteriaBuilder cb = em.getCriteriaBuilder();
	    CriteriaQuery<Tuple> query = cb.createTupleQuery();
	        
	    Root<Employee> employee = query.from(Employee.class);
	    Join<Employee, Project> project = employee.join("project");
	        
	    query
	    	.multiselect(employee, cb.count(project))
	        .groupBy(project)
	        .having(cb.ge(cb.count(project), 2));
              

	Bulk Update and Delete


		Update

			Criada atraves do metodo CriteriaBuilder.createUpdateCriteria
			Mesma coisa das outras, mas tem o metodo set

			 	UPDATE Employee e
			    SET e.salary = e.salary + 5000
			    WHERE EXISTS (SELECT p
			                  FROM e.projects p
			                  WHERE p.name = 'Release2')
   
  
		        CriteriaBuilder cb = em.getCriteriaBuilder();
		        CriteriaUpdate<Employee> update = cb.createCriteriaUpdate(Employee.class);
		        Root<Employee> employee = update.from(Employee.class);
		        
		        Subquery<Project> subquery = update.subquery(Project.class);
		        Root<Employee> employeeSub = subquery.correlate(employee);
		        Join<Employee, Project> projects = employeeSub.join("projects");           
		        
		        subquery
		            .select(projects)
		            .where(cb.equal(projects.get("name"), "Release2"));
		        
		        update
		            .set(employee.<Integer>get("salary"), cb.sum(employee.get("salary"), 5000))
		            .where(cb.exists(subquery));


		Delete

			Criado atraves do metodo CriteriaBuilder.createDeleteCriteria:

				DELETE FROM Employee e
				WHERE e.department IS NULL

				CriteriaDelete<Employee> q = cb.createCriteriaDelete(Employee.class);
				Root<Employee> emp = c.from(Employee.class);
				q.where(cb.isNull(emp.get("dept"));



Strongly Typed Query Definitions

	
		The Metamodel API

			Guarda informações de metadata das entidades gerenciadas na persitence unit
			Possivel recurar através de:

				Metamodel mm = em.getMetamodel();
				EntityType<Employee> emp_ = mm.entity(Employee.class);

		
		Strongly Typed API Overview

			Exemplo de como usar esse API para fazer querys altamente tipadas:

				SELECT emp.name, KEY(phone), VALUE(phone)
				FROM Employee emp
				JOIN emp.phones phone;

				CriteriaQuery<Object> c = cb.createQuery();
				Root<Employee> emp = c.from(Employee.class);
				EntityType<Employee> emp_ = emp.getModel();
				MapJoin<Employee,String,Phone> phone =
				    emp.join(emp_.getMap("phones", String.class, Phone.class)); //metodo join sobrecarregado para funcionar com MapAttribute
				c.multiselect(emp.get(emp_.getSingularAttribute("name", String.class)),
				              phone.key(), phone.value());


		The Canonical Metamodel


			Classes geradas no mesmo pacote e com o mesmo nome, acrescentado de um _, da entidade a qual guardam informações;
			Classe anotada com @StaticMetamodel(Entidade.class)
			Esse anotação é que faz o mapeamento entre Entidade <-> metamodel 
			Campos de tipo primitivo ou String, são representados com SingularAttribute
			Coleções ou Maps com seus tipos respectivos (ListAttribute, SetAttribute, MapAttribute, ou CollectionAttribute)

			Metamodel da classe Employee:

				@StaticMetamodel(Employee.class)
				public class Employee_ {
				    public static volatile SingularAttribute<Employee, Integer> id;
				    public static volatile SingularAttribute<Employee, String> name;
				    public static volatile SingularAttribute<Employee, String> salary;
				    public static volatile SingularAttribute<Employee, Department> dept;
				    public static volatile SingularAttribute<Employee, Address> address;
				    public static volatile CollectionAttribute<Employee, Project> project;
				    public static volatile MapAttribute<Employee, String, Phone> phones;
				}

		Using the Canonical Metamodel

			Exemplo de uso:

			CriteriaQuery<Object> c = cb.createQuery();
			Root<Employee> emp = c.from(Employee.class);
			MapJoin<Employee,String,Phone> phone = emp.join(Employee_.phones);
			c.multiselect(emp.get(Employee_.name), phone.key(), phone.value());

			------------

			SELECT emp FROM Employee
			WHERE emp.depto IN 
					(SELEC DISTINCT dept FROM Department dept
					JOIN dept.employees e JOIN e.projects project
					WHERE project.name LIKE "QA%")

				=

			CriteriaQuery<Employee> c = cb.createQuery(Employee.class);
			Root<Employee> emp = c.from(Employee.class);
			Subquery<Department> sq = c.subquery(Department.class);
			Root<Department> dept = sq.from(Department.class);
			Join<Employee,Project> project =
			    dept.join(Department_.employees).join(Employee_.projects);
			sq.select(dept.get(Department_.id))
			  .distinct(true)
			  .where(cb.like(project.get(Project_.name), "QA%"));

			c.select(emp)
			 .where(cb.in(emp.get(Employee_.dept).get(Department_.id)).value(sq));


		Generating the Canonical Metamodel





========================= CHAPTER 10 - Advanced Object-Relational Mapping =============================================================================


Table and Column Names

	SQL não é case-sentive
	Da na mesma:

		@Table(name="employee")
		@Table(name="Employee")
		@Table(name="EMPLOYEE")


	Caso necessario, pode ser definido nomes em case diferente. Para isso, use um segundo par de aspas duplas:

		@Table(name="\"Employee\"")
		@Table(name="\"EMPLOYEE\"")


	Para definir como padrão que todas os identificadores deverão ser tratados de modo literal como aparecem nas Strings, use o elemento <delimited-identifiers> no persistence.xml.


Converting Entity State
 	
 	Do JPA 2.1
 	Pode ser definido no atributo dizendo como o mesmo deve ser persistido e restaurado.
 	Implementa a interface_ AttributeConverter<X,Y>

 		X = Atributo da entidade
 		Y = JDBC type


 	Exemplo de conversor de boolean (campo da entidade) para Integer (banco de dados):

 		@Converter
		public class BooleanToIntegerConverter  implements AttributeConverter<Boolean,Integer> {
T
		    public Integer convertToDatabaseColumn (Boolean attrib) {
		        return (attrib ? 1 : 0);
		    }

		    public Boolean convertToEntityAttribute (Integer dbData) {
		        return (dbData > 0)
		    }
		}

	Conversor deve ser anotado com @Converter e deve ser listado no persitence.xml como um entidade comum.


	Declarative Attribute Conversion

		Atributo anotado deve ser equivalente ao primeiro tipo informado na declaração da classe.
		Wrappers ou primitios, tanto faz. Autoboxing funciona.

		Exemplo:

			@Convert(converter=BooleanToIntegerConverter.class)
			private Boolean bonded;


	Converting Embedded Attributes


		O conversor de exemplo mostrando acima, espera que o tipo do atributo anotado seja um boolean. Porem, caso estejamos anotando um objeto Embedable e o atrito a qual estejamos fazendo referencia seja um atributo desse objeto e nao da entidade em si, devemos usar o elemento 'attributeName' da anotação @Converter, passando nesse o nome do atributo do objecto embedable que será convertido.

		Exemplo:


			@Entity
			public class Employee {
			    // ...
			    @Convert(converter=BooleanToIntegerConverter.class, attributeName="bonded") //bonded faz referencia ao atributo da classe SecurityInfo.
			    private SecurityInfo securityInfo;
			    // ...
			}

			@Embeddable
			public class SecurityInfo {
			    private Boolean bonded;
			    // ...
			}

	Converting Collections


		Collections of basic types:

			Sem segredos. So adicionar a anotação @Converter falando a classe do conversor

			Exemplo:

				@ElementCollection
				@Convert(converter=BooleanToIntegerConverter.class)
				private List<Boolean> securityClearances;


		Collection of Embeddable


			Para coleções, sem segredo tambbem. Mesmo conceito mostrando antes. Use o elemento "attributeName" para informar qual o atributo do embedable que será convertiddo:

			Exemplo:

				@ElementCollection
				@Convert(converter=BooleanToIntegerConverter.class, attributeName="bonded")
				private List<SecurityInfo> securityClearances;


		Map of basic type


			Para maps, por padrão, se não for espeficicado nada, o que será converitod será o valor


				@ElementCollection
				@Convert(converter=BooleanToIntegerConverter.class)
				private Map<SecurityInfo, Boolean> securityClearances;

			Caso deseje que seja a chave, passe para o attributeName name, o valor "key"

				@ElementCollection
				@Convert(converter=BooleanToIntegerConverter.class, attributeName="key")
				private Map<Boolean, SecurityInfo> securityClearances;


		Map of embedable


			Para maps de embedable, prefixe o valor do atributo attributeName com o valor "key" ou "value" para especificar o que será convertido:

			Exemplos:

				Para converter o valor:

					ElementCollection
					@Convert(converter=BooleanToIntegerConverter.class, attributeName="value.bonded")
					private Map<Boolean, SecurityInfo> securityClearances;


				Para converter a chave:

					ElementCollection
					@Convert(converter=BooleanToIntegerConverter.class, attributeName="key.bonded")
					private Map<SecurityInfo, Boolean> securityClearances;



	Complex Embedded Objects

		Advanced Embedded Mappings

		 	Objetos embedados podem conter entidades e outros outros objetos embedados como atributo.
		 	Se um objeto embedado tiver um relacionamento bidirecional, o target side tem que fazer referencia ao nome qualificado do atributo owner do relacionamento.

		 	Exemplo:


		 		ContactInfo é embedado dentro de Employee

		 		@Embeddable 
		 		@Access(AccessType.FIELD)
				public class ContactInfo {
				    @Embedded
				    private Address residence;

				    @ManyToOne
				    @JoinColumn(name="PRI_NUM")
				    private Phone primaryPhone;

				    @ManyToMany @MapKey(name="type")
				    @JoinTable(name="EMP_PHONES")
				    private Map<String, Phone> phones;
				    // ...
				}

				Phone quando fizer referencia ao atributo "phones" deve usar o nome completo:

				@Entity
				public class Phone {
				    @Id private String num;

				    @ManyToMany(mappedBy="contactInfo.phones") //atributo contactInfo de Employee e phones de ContactInfo.
				    private List<Employee> employees;

				    private String type;
				    // ...
				}

		Overriding Embedded Relationships

			@AssociationOverride

				Quando se esta sobre escrevendo um atributo de um objeto embebada, deve ser usada a anotação @AttributeOverride
				Já para sobreeescrever o atributo de uma tipo embedado que é uma referencia, deve ser usada @Association.

			Exemplo.

				Pense na classe acima. Agora pense que desejamos usar ContactInfo na classe cliente. É necessario sobreescrever como alguns atributos serão gerados (PRI_NUM (por frescuro) e EMP_PHONES, para outro nome, já que não é um empregado.)



				@Entity
				public class Customer {
				    @Id int id;

				    @Embedded
				    @AssociationOverrides({
				        @AssociationOverride(name="primaryPhone",
				                             joinColumns=@JoinColumn(name="EMERG_PHONE")),
				        @AssociationOverride(name="phones",
				                             joinTable=@JoinTable(name="CUST_PHONE"))})
					@AttributeOverride(name="residence.zip", column=@Column(name="ZIP"))
				    private ContactInfo contactInfo;

				    // ...
				}




	Compound Primary Keys


		Sempre ira precisar de uma classe que represente a primary key composta

		Precisam:

			* Implementar hashcode e equals 
			* Implementar Serializable
			* Ter um construtor no-args
			* Ser publica



			Exemplo utilizado

			 _______________	
			|Employe  		|
			|_______________|
			| id	  [PK]	|
			| country [PK]	|
			| name    		|
			| salary  		|
			|_______________|


		Id Class

			Entidade:

				Cada campo da entidade que faz parte da primary key é anotado com @Id
				A entidade é anotada com @IdClass(Idclass.class)

			Primary key class:

				Ter ter os campos equivalente aos da entidade tanto em TIPO quanto no NOME DOS ATRIBUTOS.
				Depois de criada, seus valores não podem ser alterados.

			Exemplo:

				@Entity
				@IdClass(EmployeeId.class)
				public class Employee {
				    @Id 
				    private String country;
				    @Id
				    @Column(name="EMP_ID")
				    private int id;
				    private String name;
				    private long salary;
				    // ...
				}


				public class EmployeeId implements Serializable {

					private String country;
					private int id;

					public EmployeeId() {}
					public EmployeeId(String country, int id) {
						this.country = country;
					    this.id = id;
					}
					public String getCountry() { return country; }
					public int getId() { return id; }

					//HASHCODE E EQUALS
				}



		Embedded Id Class

			Entidade:

				Tem um atributo do tipo embedado que representa a chave e o anota com @EmbeddedId
				Encare essa anotação como uma junção de @Embebed e @Id

			Primary key class:

				Tipo embedable comum. Não necessario adiconar as anotações @Id no campos



			Exemplo:

				@Entity
				public class Employee {
				    @EmbeddedId private EmployeeId id;
				    private String name;
				    private long salary;

				    public Employee() {}
				    public Employee(String country, int id) {
				        this.id = new EmployeeId(country, id);
				    }

				    public String getCountry() { return id.getCountry(); }
				    public int getId() { return id.getId(); }
				    // ...
				}

				@Embeddable
				public class EmployeeId {
				    private String country;
				    @Column(name="EMP_ID")
				    private int id;

				    public EmployeeId() {}
				    public EmployeeId(String country, int id) {
				        this.country = country;
				        this.id = id;
				    }

				    // ...
				}


	Derived Identifiers

		Definição
			derived identifier
			dependent entity
			parent entity


		Parent entity 					 Depedente Entity
		_________________                _____________
		|Departament 	|			 	| Project     |
		|_______________|			 	|_____________|	
		| id	  		|				| id [PK, FK1]| FK1 = Derived Identifier
		| name    		| ---------->	| name [PK]   |
		________________|             	| start_date  |	  
							 		  	| end_date    |
										_____________ |



		Basic Rules for Derived Identifiers


		* A entidade que recebe a chave estrangeira (dependent entity) e a usa como chave primaria (derived identifier), pode ter multplos derived identifier 
		* Todos os dependent identifier devem ser setados antes que a classe seja persistida.
		* Se a classe tem multplos ids (sendo derivados ou não) ela DEVE usar uma id class, com os paramentros equivalentes a entidade
		* Ids podem ser de tipos basicos ou de referencia a entidades em relacionamentos xptoToOne
		* Se o id na entidade for e um tipo basico, o atributo equivalente na id classe deve ser do mesmo tipo
		* Se o id na entidade for um relacionamento, o atributo equivalente na id classe deve ser do tipo do id na classe referenciada (parent entity)


	Shared Primary Key

		Dependent Entity tera o mesmo tipo de id da parenet entity

			Se simples, então simples
			Se composto, então composto. Se a parent tem um id class, a dependent tambem deve ser anotada com @IdClass


			@Entity
			public class EmployeeHistory {
			    // ...
			    @Id
			    @OneToOne
			    @JoinColumn(name="EMP_ID")
			    private Employee employee;
			    // ...
			}

		Esse caso burla a lei de que a entidade deve ter campos com mesmo nome e tipo da id class.



		Para facilitar o acesso, caso deseje que entidade tenha um atributo separado para o id (mas que tambem é a chave estrangeira), é possivel utilizar @MapsId


		 	@Entity
			public class EmployeeHistory {
			    // ...
			    @Id
			    int empId;

			    @MapsId
			    @OneToOne
			    @JoinColumn(name="EMP_ID")
			    private Employee employee;
			    // ...
			}

		Ao persistir uma classe mapeada dessa forma, sempre defina o valor do atributo do relacionamento. Ele é o id da classe. O identificador (atributo anotado com @Id), sera preenchido automaticamente ao persistir a classe.

		NÃO SERAO GERADOS DOIS CAMPOS DE ID. Esse mapeamento é apenas para facilitar o acesso na classe.

		DUVIDA:
			O que acontece caso a classe tenha 2 chaves estrangeiras utilizadas como id, como diferencia-las no @MapsId
			Exemplo:


			@Entity
			public class EmployeeHistory {
			    // ...
			    @Id
			    int empId;

			   	@Id
			    int historyId;

			    @MapsId
			    @OneToOne
			    @JoinColumn(name="EMP_ID")
			    private Employee employee;


			    @MapsId
			    @OneToOne
			    @JoinColumn(name="HIS_ID")
			    private History history;
			    // ...
			}

		REPOSTA:

			@MapsId pode receber um String falando de qual atributo da classe se trata.
			Ficaria assim:

				//...
				@MapsId("empId")
				//...
				@MapsId("historyId")


	Multiple Mapped Attributes


		Caso a entidade tenha um identificador composto, uma boa forma de mapea-lo é utilizando @IdClass.
		Regras:

			A idclass deve ter atributos com o mesmo nome
			Se o id da entidade é um tipo basico, a idclass deve ter o mesmo tipo basico mapeado.
			Se o id da entidade é um relacionamento, a idclass deve ter um tipo basico igual ao tipo do id do relaciomaneto

		Exemplo:

			@Entity
			@IdClass(ProjectId.class)
			public class Project {

			    @Id private String name;

			    @Id
			    @ManyToOne
			    @JoinColumns({
			        @JoinColumn(name="DEPT_NUM",
			                    referencedColumnName="NUM"),
			        @JoinColumn(name="DEPT_CTY",
			                    referencedColumnName="COUNTRY")})
			    private Department dept;
			    // ...
			}

			A idclass (ProjectId) de ter uma String chamada 'name' e um campo chamado 'dept' do mesmo tipo do id de Department.

			public class ProjectId implements Serializable {
			    private String name;
			    private DeptId dept;

			    public ProjectId() {}
			    public ProjectId(DeptId deptId, String name) {
			        this.dept = deptId;
			        this.name = name;
			    }
			   // ...
			}

			Nesse caso, a chave primaria de Department é um tipo composto mapeado com uma IdClass chamada DeptId. Sendo assim, o atributo 'dept' de ProjectId é do tipo DeptId (equivalente ao id de Department).


			public class DeptId implements Serializable {
			    private int number;
			    private String country;

			    public DeptId() {}
			    public DeptId (int number, String country) {
			        this.number = number;
			        this.country = country;
			    }
			    // ...
			}

	Using EmbeddedId

		É possivel fazer o mesmo tipo de mapeamento utilizando EmbeddedId


		@Entity
		public class Project {

		    @EmbeddedId private ProjectId id;

		    @MapsId("dept")
		    @ManyToOne
		    @JoinColumns({
		       @JoinColumn(name="DEPT_NUM", referencedColumnName="NUM"), //referece a Department.DeptId.number
		       @JoinColumn(name="DEPT_CTRY", referencedColumnName="CTRY")})//referece a Department.DeptId.country
		    private Department department;
		    // ...
		}

		@Embeddable
		public class ProjectId implements Serializable {
		    @Column(name="P_NAME")
		    private String name;
		    @Embedded
		    private DeptId dept;

		    // ...
		}

		@Entity
		public class Department {
		    
		    @EmbeddedId
		    private DeptId id;
		    
		    @OneToMany(mappedBy="department")
    		private List<Project> projects;

		    // ...
		}

		@Embeddable
		public class DeptId implements Serializable {
		    @Column(name="NUM")
		    private int number;
		    @Column(name="CTRY")
		    private String country;
		    // ...
		}

		LEMBRE-SE:

			@Id somente na Entidade.
			@EmbeddedId na entidade e tambem no tipo embebado caso esse tambem tenha referencia a um id composto
			Classe referenciada na anotacao Idclass. Nao tem campos anotados com @Id. Somente tem campos com o mesmo nome da entidade. 

	
	Advanced Mapping Elements

		Read-Only Mappings

			Util quando as entidades ja existem na base de dados e serao usadas apenas para consulta:

				@Entity
				public class Employee {
				    @Id
				    @Column(insertable=false)
				    private int id;
				    @Column(insertable=false, updatable=false)
				    private String name;
				    @Column(insertable=false, updatable=false)
				    private long salary;

				    @ManyToOne
				    @JoinColumn(name="DEPT_ID", insertable=false, updatable=false)
				    private Department department;
				    // ...
				}

			Pontos:

				Id nao precisa ser definido como não atualizavel pq isso já ilegal de qualquer forma.
				Caso a entidade seja alterada, ao termino da transacao ou quando a mesma sofre um flush, seu estado nao sera persistido.

				DUVIDA:
					No persistence context, como fica o estado da entidade? Igual ao banco de dados ou a entidade alterada?
					Pela explicação, acredito que o persistence context, seja alterado.
					TESTAR!!

		Optionality


			Disponivel em @Basic, @ManyToOne, e @OneToOne
			Utilizado para informar se um atributo pode ou não ser nullo ao persistir.
			Valor padrão é true.
			Comportamento padrão quando essa constraint é quebrada não esta especificado.

			Exemplo de uso:

				@Entity
				public class Employee {
				    // ...
				    @ManyToOne(optional=false)
				    @JoinColumn(name="DEPT_ID", insertable=false, updatable=false)
				    private Department department;
				    // ...
				}

	Advanced Relationships


		Using Join Tables

			Em relacionamentos @ManyToOne é desnecessario o uso de tabelas de relaciomento. A entidade do lado many pode muito bem ter em sua tabela o o id do lado one. POREM, caso deseje, pode usar a anotação @JoinTable nesse tipo de relacionamento para criar a tabela de junção.

			   /* 
			    * Cria um tabela para fazer o mapeamento entre funcionario e departamento.
			    * Nesse tipo de relacionamento, essa tabela nao é necessaria pois o funcionario poderia ter
			    * em sua tabela o id do departmento a qual pertence.
			    */ 
				@JoinTable( name="employee_department",
		                joinColumns=@JoinColumn(name="employee_id", nullable=false),
		                inverseJoinColumns=@JoinColumn(name="dept_id"))
		    	@ManyToOne
				private Department department;

		Compound Join Columns


			Ao usar mapeameto com chaves composta, use a anotação @JoinColumns
			Usa sem especificar as colunas é invalido >> @JoinColumns({}) invalido
			Use o atributo 'referencedColumnName' da anotação @JoinColumn para especificar qual campo vc esta falando.
			No Hibernate, caso esse atributo nao seja especificado, ele pega conforme a ordem de definição das chaves:

				1º campo da chave composta >> 1º anotação @JoinColumn
				2º campo da chave composta >> 2º anotação @JoinColumn

			MAS, não sei se esse é o comportamento padrão. Por boas praticas e seguraça, use 'referencedColumnName' para especificar qual coluna vc esta sobreeescrendo.

			Fora a anotação @JoinColumns, anotações @XptoToMany tem um atributo 'joinColumns' que funciona da mesma forma.

			@Entity(name="company")
			public class Company {
			    
			    @EmbeddedId
			    private CompanyIdentification identification;
				   //..

			    @ManyToOne
			    @JoinColumns({
			        @JoinColumn(name = "m_name", referencedColumnName="name"),
			        @JoinColumn(name = "m_cnpj", referencedColumnName="cnpj")})
			    private Company matriz;

			    //...
			}

			@Embeddable
			public class CompanyIdentification implements Serializable {

			  	//...

			    @Column(columnDefinition="VARCHAR(14)")
			    private String cnpj;

			    @Column(columnDefinition="VARCHAR(100)")
			    private String name;

			    //...
			}

	Orphan Removal

	 	Disponivel apenas nos relacionamentos @OneToXpto
	 	Equivalente a definir Cascade delete no relacionamento.
	 	Quando o pai for removido, o filho tambem será
	 	Quando o filho for removido da lista do pai, o mesmo sera deleteado.


	 		@Id private int id;
		    @OneToMany(orphanRemoval=true)
		    private List<Evaluation> evals;

		Em mapas, aplicado somente ao valor.


	Mapping Relationship State


		Diferentemente do que acontece com as tabelas, ao usar as tabelas de junção de entidades, não é possivel adicionar campos na mesma.
		Para isso, é necessario criar uma entidade que tem relacionamento @ManyToOne para as entidades originais e essas terão o inverso.


			@Entity
			@Table(name="EMP_PROJECT")
			@IdClass(ProjectAssignmentId.class)
			public class ProjectAssignment {
			    @Id
			    @ManyToOne
			    @JoinColumn(name="EMP_ID")
			    private Employee employee;

			    @Id
			    @ManyToOne
			    @JoinColumn(name="PROJECT_ID")
			    private Project project;

			    @Temporal(TemporalType.DATE)
			    @Column(name="START_DATE", updatable=false)
			    private Date startDate;
			    // ...
			}

			public class ProjectAssignmentId implements Serializable {
			    private int employee;
			    private int project;
			    // ...
			}


			@Entity
			public class Employee {
			    @Id private int id;
			    // ...
			    @OneToMany(mappedBy="employee")
			    private Collection<ProjectAssignment> assignments;
			    // ...
			}

			@Entity
			public class Project {
			    @Id private int id;
			    // ...
			    @OneToMany(mappedBy="project")
			    private Collection<ProjectAssignment> assignments;
			    // ...
			}




	Multiple Tables


		Coisa de maluco.
		Usado quando deseja que uma entidade seja persistida em mais de uma tabela.
		Use e anotação @SecondaryTable e @SecondaryTables para definir quais são as tabelas usadas.

		Exemplo simples:

			Serão geradas duas tabelas "EMP" e "EMP_ADDRESS". Os campos que a anotação @Column define o valor, serão guardados na tabela "EMP_ADDRESS"
			A anotação @SecondaryTable ainda define qual sera o nome da chave primaria na nova tabela. No caso "EMP_ID". Caso esse valor não fosse fornecido, o nome da chave primaria na tabela secundaria seria o mesmo da tabelam primaria.

			@Entity
			@Table(name="EMP")
			@SecondaryTable(name="EMP_ADDRESS",
			    pkJoinColumns=@PrimaryKeyJoinColumn(name="EMP_ID"))
			public class Employee {
			    @Id private int id;
			    private String name;
			    private long salary;
			    @Column(table="EMP_ADDRESS")
			    private String street;
			    @Column(table="EMP_ADDRESS")
			    private String city;
			    @Column(table="EMP_ADDRESS")
			    private String state;
			    @Column(name="ZIP_CODE", table="EMP_ADDRESS")
			    private String zip;
			    // ...
			}


		Outro exemplo:

			Nesse caso, os atributos do objecto embedado serão persistidos em outra tabela.

			@Entity
			@Table(name="EMP")
			@SecondaryTable(name="EMP_ADDRESS", pkJoinColumns=@PrimaryKeyJoinColumn(name="EMP_ID"))
			public class Employee {
			    @Id private int id;
			    private String name;
			    private long salary;
			    @Embedded
			    @AttributeOverrides({
			        @AttributeOverride(name="street", column=@Column(table="EMP_ADDRESS")),
			        @AttributeOverride(name="city", column=@Column(table="EMP_ADDRESS")),
			        @AttributeOverride(name="state", column=@Column(table="EMP_ADDRESS")),
			        @AttributeOverride(name="zip", column=@Column(name="ZIP_CODE", table="EMP_ADDRESS"))
			    })
			    private Address address;
			    // ...
			}


		Outro exemplo, com mais de uma tabela secundaria e com chave primaria composta:

			Serão criadas 3 tabelas:

				* Employee (Primaria)
				* ORG_STRUCTURE (Secundaria)
				* EMP_LOB (Secundaria)

			Já que a entidade primaria tem uma chave primaria composta com 2 campos, são usadas duas anotações @PrimaryKeyJoinColumn na definição de cada tabela secundaria, em ambos os casos definindo a chave o nome da coluna com a chave primaria para "COUNTRY" e "EMP_ID"

			@Entity
			@IdClass(EmployeeId.class)
			@SecondaryTables({
			    @SecondaryTable(name="ORG_STRUCTURE", pkJoinColumns={
			        @PrimaryKeyJoinColumn(name="COUNTRY", referencedColumnName="COUNTRY"),
			        @PrimaryKeyJoinColumn(name="EMP_ID", referencedColumnName="EMP_ID")}),
			    @SecondaryTable(name="EMP_LOB", pkJoinColumns={
			        @PrimaryKeyJoinColumn(name="COUNTRY", referencedColumnName="COUNTRY"),
			        @PrimaryKeyJoinColumn(name="ID", referencedColumnName="EMP_ID")})})
			public class Employee {
			    
			    @Id 
			    private String country;
			    @Id
			    @Column(name="EMP_ID")
			    private int id;

			    @Basic(fetch=FetchType.LAZY)
			    @Lob
			    @Column(table="EMP_LOB")
			    private byte[] photo;
			     
			    @Basic(fetch=FetchType.LAZY)
			    @Lob
			    @Column(table="EMP_LOB")
			    private char[] comments;

			    @ManyToOne
			    @JoinColumns({
			        @JoinColumn(name="MGR_COUNTRY", referencedColumnName="COUNTRY",
			                    table="ORG_STRUCTURE"),
			        @JoinColumn(name="MGR_ID", referencedColumnName="EMP_ID",
			                    table="ORG_STRUCTURE")
			    })
			    private Employee manager;
			    // ...
			}


	Inheritance

		Mapped Superclasses	

			Super classe do relacionamento
			Agrupamento do comportamento generico das classes filhas.
			Não pode ser persistida
			Não pode ser usada em queries
			NAO PODE SER USADA PARA RELACIONAMENTOS.
			Nao pode usar @Table
			Podem ou não ser abstratas.


		Modelo:

			@Entity
			public class Employee {
			    @Id private int id;
			    private String name;
			    @Temporal(TemporalType.DATE)
			    @Column(name="S_DATE")
			    private Date startDate;
			    // ...
			}

			@Entity
			public class ContractEmployee extends Employee {
			    @Column(name="D_RATE")
			    private int dailyRate;
			    private int term;
			    // ...
			}
			@MappedSuperclass //NÃO PODE SER USADA PARA QUERIES, COMO TARGET DE UM RELACIONAMENTO E NEM PERSISTIDA. 
			public abstract class CompanyEmployee extends Employee {
			    private int vacation;
			    // ...
			}

			@Entity
			public class FullTimeEmployee extends CompanyEmployee {
			    private long salary;
			    private long pension;
			    // ...
			}

			@Entity
			public class PartTimeEmployee extends CompanyEmployee {
			    @Column(name="H_RATE")
			    private float hourlyRate;
			    // ...
			}


	Transient Classes in the Hierarchy

		Classes transientes são aquelas que não tem nem anotação de entidade ou de super_ classe mapeada.
		Seu estado não sera persistido no banco de dados, mas todas as leis de heranca do java se aplicam normalmente


	Abstract and Concrete Classes

		Entidades, Super classes mapeadas e transientes classes podem ser abstratas ou concretas.
		Pode parecer estranho ter uma entidade abstrata. Na sua cabeça ela deveria ser uma @MappedSuperclass, mas caso esse fosse o caso, suas queries não poderiam ser feitas com essa classe.
		Caso uma entidade abstrata seja utilizada em queries, o retorno será um conjunto (ou somente uma) de suas classes concretas.


	Inheritance Models

		DUVIDA:
			A estrategia de herança pode ser uma até uma parte da arvore e dessa parte pra baixo ser outra?
			RESPOSTA:
				Sim, entre os tipos suportados obrigatoriamente pela aplicação (Single table e Join)


		Single-Table Strategy

			Quem será peristidos:
				Entidades concretas.


			Como funciona?

				O tipo do id deve ser compativel em todas as classes, já que apenas uma coluna ira persitir o valor de todas.

				Para todas as entidades na arvore de herança, é gerado uma tabela contendo todos os campos mapeados (isso não inclui os da transientes entities)

			Como é gerada a tabela
					Uma tabela com todas as colunas. Se a entidade X tem o campo 'a', no momento que a entidade Y (que não herdam é irmã de X) for persistida, o valor da sua coluna 'a' ficara nulo.

			DiscriminatorColumn
					Definida na classe mãe que recebe a anotação @Inheritance e diz qual será o nome da coluna usada para diferenciar qual tipo de entidade esta persistida na linha.
					Se nenhum nome for informado, a coluna será criada com o seu valor padrão 'DTYPE'. Os tipos permitdos para essa coluna são STRING, INTEGER e CHAR
			
			DiscriminatorValue

				Em que classe vai?
					Cada entidade deve receber essa anotação e deve informar qual o seu identificador.
					NAO VAI EM:
						Abstract entity
						MappedSuperclass
						Transient classes

				Se nao for usado, o valor padrão varia de tipo para tipo.
				 String: Quando a coluna for do tipo String, o nome simple da classe será utilizado.
				 Integer: Será gerado um valor pelo provider. Ao usar esse tipo de identificador, ou defina o valor para cada entidade ou não defina nenhum. Caso seja informado apenas alguns, há o risco de colisão com os valores gerados pelo provider.

		Codigo:

			@Entity
			@Table(name="EMP")
			@Inheritance
			@DiscriminatorColumn(name="EMP_TYPE")
			public abstract class Employee { ... }

			@Entity
			public class ContractEmployee extends Employee { ... }

			@MappedSuperclass
			public abstract class CompanyEmployee extends Employee { ... }

			@Entity
			@DiscriminatorValue("FTEmp")
			public class FullTimeEmployee extends CompanyEmployee { ... }

			@Entity(name="PTEmp")
			public class PartTimeEmployee extends CompanyEmployee { ... }



		Joined Strategy

			Quem sera peristidos: Entidades concretas ou abstratas
			
			A chave primaria de uma subclass_ age tambem como chave estrangeira para a MappedSuperclass.

			Como funciona?

				Toda entidade gera uma tabela.
				A tabela gerada terá os atributos definidos naquela entidade.
				As entidades/tabelas filhas terão apenas seus proprios valores definidos.
				Ao persistir uma classe filha sua PK, terá o mesmo valor que o pai, e na classe filha funciona tambem como uma FK apontando para o id do pai.


			 @DiscriminatorColumn e  @DiscriminatorValue

			 	A definição dessas anotações acontece da mesma forma que na estrategia de tabela unica. 

			 		@DiscriminatorColumn na classe que define a estrategia
			 		@DiscriminatorValue nos filhos

			 	A diferença esta na tabela gerada:

			 		Somente a tabela pai (nesse caso, Employee é uma entidade) e ela tera a coluna de descriminação de tipo.



			Codigo:

				@Entity
				@Table(name="EMP")
				@Inheritance(strategy=InheritanceType.JOINED)
				@DiscriminatorColumn(name="EMP_TYPE", discriminatorType=DiscriminatorType.INTEGER)
				public abstract class Employee { ... }

				@Entity
				@Table(name="CONTRACT_EMP")
				@DiscriminatorValue("1")
				public class ContractEmployee extends Employee { ... }

				@MappedSuperclass
				public abstract class CompanyEmployee extends Employee { ... }

				@Entity
				@Table(name="FT_EMP")
				@DiscriminatorValue("2")
				public class FullTimeEmployee extends CompanyEmployee { ... }

				@Entity
				@Table(name="PT_EMP")
				@DiscriminatorValue("3")
				public class PartTimeEmployee extends CompanyEmployee { ... }


		Table-per-Concrete-Class Strategy


			É gerado uma tabela por entidade concreta.
			Todo estado herdado é sempre criado (e possivelmente repetido) em casa tabela.



		Mixed Inheritance



========================= CHAPTER 11 - Advanced Queries ==============================================================================================



	Ao usar uma @NativeNamedQuery, é possivel usar o metodo EntityManager.createNamedQuery, que recebe o nome da query e seu tipo de retorno. Esse metodo retorna um TypedQuery
	Ao não usar @NativeNamedQuery, e sim a String literal representando o SQL, o tipo de retorno do metodo EntityManager.createNativeQuery, mesmo o metodo sobrecarregado que recebe uma class_ retorna apenas um Query.


	Entidades retornadas por NamesQuery TAMBEM são gerenciadas pelo persistence context. O que significa que quando a transação for comitada, as entidades no persistence context serão persistidas. O que significa que caso sua querie não tenha retornado a entidade completa (campos null ou zero), quando a entidade for persistida, o EntityManager NAO TEM COMO SABER que aquele valor não "é original" da entidade. Para ele,essa entidade foi atualizada, e ele, como bom rapaz que é, ira persistir essa entidade e sobreescrever os valores validos do banco de dados.


	Result Mapping

		Quando a native query retorna campos com nomes diferentes da entidade, com mais campos, etc, essa anotação deve ser usada para fazer o mapeamento entre banco de dados e entidade


		O codigo abaixo faz o mapeamento entre os campos renomeados da entidade Employee e UM campo de retornado do Department

		SELECT e.id as identificador, e.name as nome, [...], d.name as name FROM Employee e, Department d;


		@SqlResultSetMapping(name="EmployeeResultSetMapping", 
		    entities=
		    	@EntityResult(entityClass = Employee.class, 
		    		fields=@FieldResult(name="id", column="identificador"),
		    		fields=@FieldResult(name="name", column="nome")),
		    columns=@ColumnResult(name="nome")
		)

		Apenas os campos retornados com nome diferente precisam ser mapeados usando @FieldResult.

		Pode ser muito mais simples, caso por exemplo o tipo retornado seja a entidade completa e os campos da tabela tenham os mesmos nomes mapeados na mesma.

		SELECT * FROM Employee;

		@SqlResultSetMapping(name="EmployeeResultSetMapping", entities=@EntityResult(entityClass = Employee.class))

		//Equivalente a escrever uma NamedQuery e ao usar o EntityManager, passar o tipo junto.


		Para usar:

			em.createNativeQuery(QUERIE NATIVA, NOME UNICO DO RESULT SET MAPPING)


	
		A query por retornar mais de uma entidade. Caso isso seja feito, o atributo "entities" da anotacao recebe um array. Então, passe todos os tipos retornados ali.


			Exemplo: 
			
				SELECT emp_id, name, salary, manager_id, dept_id, address_id, id, street, city, state, zip
				FROM emp, address
				WHERE address_id = id

				@SqlResultSetMapping(
				    name="EmployeeWithAddress",
				    entities={@EntityResult(entityClass=Employee.class),
				              @EntityResult(entityClass=Address.class)}
				)



	Parameter Binding

		Queries nativas aceitam apenas o bidding posicional dos parametros, não suportando named bidding.


	Defining and Executing Stored Procedure Queries


========================= CHAPTER 12 - Other Advanced Topics ==========================================================================================

	
Lifecycle Callbacks
	

	Lifecycle Events


		PrePersist and PostPersist
		
			Ambos tem a execução relacinada ao metodo EntityManager.persiste.
			Caso EntityManager.merge tenha sido invocado usando uma entidade que ainda não pertence o persitence context, o evento de PrePersist tambem será chamado.
			Entidades em relacionados com o cascade com PERSISTE ativado, tambem terão seus eventos chamados.
			PostPersist não garante que entidade foi persistida corretamente no banco, pois entre a chamada desse metodo e o fim da transação, a mesma pode ter sido rolled back;


		PreRemove and PostRemove

			Relacionados ao metodo EntityManager.remove.
			Caso a entidade esteja em um relacionamento com o REMOVE no cascade, seus eventos tambem serão chamados.
			Da mesma forma que PostPersist, a chamada do evento PostRemove não garante que a entidade será removida pois entre a chamada desse metodo (que acontece após o SQL ser enviado para o banco) a execução do SQL no banco, a trasação pode ser rolled back.



		PreUpdate and PostUpdate

			Relacionado a updates na entidade.
			Diverge entre as implementações o que acontece caso uma entidade seja persistida e atualizada dentro da mesma transação.

				* Em alguns apenas uma ação de update ocorre
				* Em outros acontecera uma para cada alteração.

			O PostUpdate é executado apenas uma vez.

			A mesma coisa que acontece com o outros PostXpto tambem pode acontece com esse evento do ciclo de vida.



		PostLoad

			Acontece após uma entidade ser recuprada do banco dedos, seja isso atráves de uma querie, um refresh ou um acesso lazy a um relacionamento.
			Relacionamentos com o cascade em REFRESH serão afetados, porem, como em todos os outros caso, a ordem de invocacao desses metodos não é garantida.


		Callback Methods

			Metodos finais ou estaticos não sao validos.
			Não deve receber parametro
			Não precisam ser publicos.
			Deve retornar void
			Não podem lançar checked exception.
			Caso seja lançado alguma runtime exception, alem de não ser executado mais nenhum evento de ciclo de vida, a transação será marcada para roll back.
			Apenas um metodo anotado para o mesmo evento de ciclo de vida.
			Não podem usar o entityManager



	Entity Listeners

		Pode ser aplicado a mais de uma entidade
		Uma entidade pode ter mais de um Entity Listener
		Como acontece nas entidades, apenas um metodo pode ser anotado com um evento especifico.
		O metodo DEVE receber um parametro do tipo da classe anotada, sua superclasse ou uma interface_ que a mesma implemente.
		Deve ter um construtor sem argumentos.

	Attaching Entity Listeners to Entities

		O Listerner não recebe anotação nenhuma
		O Listerner apenas é listado na anotação @EntityListener que deve ir na classe.
		Caso haja mais de um, serão chamados pela ordem declarada.
		Apos a execução dos eventos do listener, caso a entidade tambem declare algum evento, o mesmo será chamado.
		Caso algum desses metodos lance uma exception, a chain de execução será interrompida e a transação sera marcada como roll back

	Default Entity Listeners

		São declarados no persistence.xml e são transversais a todo o persistence unit.
		São chamados antes dos listeners listados na anotação @EntityListener.
		Para excluir sua execução, a entidade pode usar a anotação @ExcludeDefaultListeners.


	Inheritance and Lifecycle Events

		Inheriting Callback Methods

			Podem ser definidos tanto em classes concretas como tambem em abstratas.
			Se a classe pai define um metodo do ciclo de vida, esse metodo será chamado ANTES do metodo do classe filha, se esse definir algum.
			Valido para entidades ou MappedSuperclass.
			Caso a entidade X, filha da entidade Y, defina um metodo com mesmo nome que a pai (digamos callMeBaby()). O metodo X.callMeBaby() sera chamado no lugar de Y.callMeBaby().

		Inheriting Entity Listeners

			Como acontece com metodos de Callback, tambem são herdados e tambem são chamados antes.
			Caso a classe filha defina seus proprios listeners, os mesmos serão ADICIONADOS a chain de execução da classe pai.
			Caso a classe pai decida que não quer que seus listeners sejam herdados, ela deve usar a anotação @ExcludeSuperclassListeners, que irá remover a "herança" de listeners.


		Lifecycle Event Invocation Order

			1 - Listeners Default, definidos no arquivo de XML
			2 - EntityListeners mapeados para cada entidade ou MappedSuperclass_ na arvode de heranca da entidade.
			3 - Repete o passo 2 para cada classe na arvore de herança, de cima pra baixo.
			4 - Procura na entidade ou MappedSuperclass_ mais alta na arvore de herança metodos anotados para o evento X.
			5 - Repete o passo 4 para cada classe na arvore de herança, de cima para baixo. 


	Locking

		Optimistic vs Pessimistic

			Na versão otimista, ao comitar a transação é verificado a @Version da entidade. Caso haja divergencia, é lançado uma exception.
			Na versão pessimista, o registro é bloqueado e não pode ser escrito e/ou lido

		Optimistic Locking

			A parte de ser otimista presume que apenas uma transação irá mexer na entidade.
			Esse modo de lock apenas ira travar a entidade no momento em que as mudanças forem ser salvas no banco de dados, provavelmente sendo isso no commit da transação.
			Se no momento em que as alterações forem ser persistidas houver disparidade entre o estado original dessa entidade na transação e o estado da mesma no banco de dados, será lançado um OptimisticLockException.


	Versioning

		Campo anotado com @Version usando pelo provider para saber quando a entide foi alterada.
		Usado pelo Optimistic locking para comparar as versoes da entidade.
		Pode ser int, long, short ou qualquer um dos seus wrappers. Tambem pode ser java.sql.Timestamp.
		Valor será persistido no banco de dados.
		Updates na entidade alteram esse valor automaticamente.
		Bulk updates devem alterar esse valor manualmente.

			UPDATE Employee e
			SET e.salary = e.salary + 1000, e.version = e.version + 1
			WHERE EXISTS (SELECT p
			              FROM e.projects p
			              WHERE p.name = 'Release2')


	Advanced Optimistic Locking Modes


		Todos os metodos abaixo podem receber um parametro adicional informando qual sera o modo de lock utilizada.
		Parametro do tipo LockModeType

			EntityManager.lock()	
			EntityManager.refresh()	
			EntityManager.find()	
			Query.setLockMode()		

		Tirando o metodo de Query, que deve ser executado em uma transação, os outros metodos devem ser chamados no escopo de uma transação.

		
		Optimistic Read Locking

			LockModeType.OPTIMISTIC = LockModeType.READ
			É o modo padrão explicado antes.
			Caso uma entidade tenha sido alterado durante por outra transação durante a execução da transação primaria, alguma das duas irá falhar ao comitar as mudanças.


		
		Optimistic Write Locking

			LockModeType.OPTIMISTIC_FORCE_INCREMENT = LockModeType.WRITE
			Faz o mesmo que anterior, mas, o usuario alterando ou não a entidade, o numero de versionamento da mesma será incrementada.
			Util para forçar mudanças na entidade não dona do relacionamento quando a entidade que é dona do relacionamento sofre alguma alteração.

				Employee tem um relacionamento unidirecional OneToMany com Uniform.
				Quando um uniforme é adicionado a lista do empregado, Employee não sofre alteração, então seu numero de versão não será incrementado.
				Ao usando o OPTIMISTIC_FORCE_INCREMENT, o incremento da versão é transversal ao relaciomaneto.


		Pessimistic Write Locking

			Bloqueia a entidade para que nenhuma outra aplicação possa altera-la ou ler a mesma.
			Constante LockModeType.PESSIMISTIC_WRITE
			O Optimistic Locking (@Version) sempre TAMBEM é validado, mesmo ao usar esse modo.


		Pessimistic Read Locking

			Bloqueia a entidade para que nenhuma outra aplicação possa altera-la
			Usando quando prentende-se ler a entidade diversas vezes.
			Constante LockModeType.PESSIMISTIC_READ

		Pessimistic Forced Increment Locking

		 	Mesma coisa do LockModeType.OPTIMISTIC_FORCE_INCREMENT, porem sendo um LockModeType.PESSIMISTIC_WRITE


		Transaction Exceptions
			
			OptimisticLockException 
				Acontece ao usar Optimistic Locking
				Faz a transação roll back

			PessimisticLockException
				Acontece ao usar Pessimistic Locking
				Faz a transação ser marcada para roll back
				

			LockTimeoutException
				Acontece ao usar Pessimistic Locking
				NAO FAZ a transação ser marcada para roll back


		Pessimistic Scope

			Ao usar um lock do tipo pessimista, apenas as tabelas relacionadas a entidade são travadas. Isso acontece para evitar problemas de escalonamento e deadlock.
			Com isso, os relacionamentos que a entidade é dona ou relacionamentos one-to-many NAO serão travados.
			Caso deseja que isso aconteça, use a propriedade PessimisticLockScope.EXTENDED

			Deve ser informada no Map passado junto da query:

				Map<String,Object> props = new HashMap<String,Object>();
				props.put("javax.persistence.lock.scope",PessimisticLockScope.EXTENDED);
				em.find(Employee.class, 42, LockModeType.PESSIMISTIC_WRITE, props);
			
		
		Pessimistic Timeouts

			LockTimeoutException só acontece com Pessimistic Locking
			É possivel definir qual será o tempo que o lock deve ser aguardado.
			

			Map<String,Object> props = new HashMap<String,Object>();
			props.put("javax.persistence.lock.timeout",5000);
			em.find(Employee.class, 42, LockModeType.PESSIMISTIC_WRITE, props);
			
					OU

			TypedQuery<Employee> q = em.createQuery(
			        "SELECT e FROM EMPLOYEE e WHERE e.id = 42",
			        Employee.class);
			q.setLockMode(LockModeType.PESSIMISTIC_WRITE);
			q.setHint("javax.persistence.lock.timeout",5000);


		Recovering From Pessimistic Failures


			As vezes pode não ser possivel adquirir o lock para a entidade.

			LockTimeoutException
				Nao conseguiu pegar o lock, mas NAO é fatal, nao faz a transacao rollback

			PessimisticLockException
				Nao conseguiu pegar o lock, e é fatal, nao faz a transacao rollback.

			Alguns comportamentos aceitaveis seria recuperar a exception e restaurar o estado das entidades.

Caching

		Sorting Through the Layers

			Interface javax.persistence.Cache
			Recuperavel através de EntityManagerFactory.getCache()
			Suporte não obrigatorio.
			Cache.evict e suas variações.
			Possivel excluir um objeto especifico (Cache.evict(Class, id)), um tipo completo (Cache.evict(Class)) ou todas as instancias (Cache.evictAll())
			Não recomendavel excluir apenas um objeto ou um tipo especifico; Pode causar problemas de inconsistente.

		Static Configuration of the Cache

			Classe ou unit
				O modo de cache pode ser definido em nivel de classe ou individualmente para classes;

			Para definir globlamente, no persiste.xml use a tag shared-cache-mode. Para criação dinamica, se a hint javax.persistence.sharedCache.mode. Os valores abaixo são validos. 
			
			NOT_SPECIFIED
				Padrão. Será usado o modo padrão definido pelo provider. Alguns dependem mais ou menos em cache.

			ALL
				Irá adicionar todas as classes ao cache.

			NONE
				Não irá adicionar nenhuma.

			DISABLE_SELECTIVE
				Ira adicionar todas, mas as classes anotadas com @Cacheable(false) serão excluidas.
				Aplicavel a todas as subclasses, mas a mesmas podem sobreescrever;

			ENABLE_SELECTIVE
				Não ira adicionar nenhuma classe, apenas as anotadas com @Cacheable(true) ou @Cacheable [ja que o value padrão é true]

		
		Dynamic Cache Management


			É possivel definir quando uma classe será lida/inserida ou não no cache.
			Para isso, ao executar uma query, use as queries hint abaixo;


			javax.persistence.cache.retrieveMode  	
				Diz se uma entidade deve ou não ser recuperada diretamente do cache. O valor deve ser do tipo da ENUM CacheRetrieveMode. Seus valores são:

					USE: Padrão. A entidade será recuperada do cache.
					BYPASS: O Cache será ignorado e o valor da entidade será recuperado do banco de dados.


			javax.persistence.cache.storeMode		
				Diz se uma entidade deve ou não ser guardada no cache. O valor deve ser do tipo da ENUM CacheStoreMode. Seus valores são:

					USE: Padrão. A entidade sera salva no cache.
					BYPASS: O cache não será atualizado com o valor da entidade.
					REFRESH: Irá atualizar o valor da entidade. Util para quando mais de uma aplicação usa o mesmo banco de dados.


			Exemplo de uso:

				public List<Stock> findExpensiveStocks(double threshold) {
				    TypedQuery<Stock> q = em.createQuery(
				        "SELECT s FROM Stock s WHERE s.price > :amount",
				        Stock.class);
				    q.setHint("javax.persistence.cache.retrieveMode",
				                   CacheRetrieveMode.BYPASS);
				    q.setHint("javax.persistence.cache.storeMode",
				                   CacheStoreMode.REFRESH);
				    q.setParameter("amount", threshold);
				    return q.getResultList();
				}


			Por padrão, o metodo refresh não atualiza o valor da entidade no cache. Isso pode ser alterado ao usar:

				HashMap props = new HashMap();
				props.put("javax.persistence.cache.storeMode",
				          CacheStoreMode.REFRESH);
				em.refresh(emp, props);




========================= CHAPTER 13 - XML Mapping Files ==========================================================================================



Disabling Annotations


	
	xml-mapping-metadata-complete

	
		Ao usar o elemento xml-mapping-metadata-complete TODAS as annotations, de todas as classes não serão processadas; Apenas o mapeamento feito no xml será considerado. 
		
			<entity-mappings>
			    <persistence-unit-metadata>
			        <xml-mapping-metadata-complete/>
			    </persistence-unit-metadata>
			</entity-mappings>



	metadata-complete

		Ao mapear uma entidade os dados do XML e das anotações serão complementares, sendo que o XML sobrepoe qualquer anotação.
		Para desativar a mapeamento vi anotação EM UMA CLASSE ESPECIFICA, use o atributo metadata-complete;


		Exemplo:

			@Entity
			public class Employee {
			    @Id private int id;
			    @Column(name="EMP_NAME")
			    private String name;
			    @Column(name="SAL")
			    private long salary;
			    // ...
			}
			orm.xml snippet:
			<entity-mappings>
			    ...
			    <entity class="examples.model.Employee"
			            metadata-complete="true">
			        <attributes>
			            <id name="id"/>
			        </attributes>
			    </entity>
			    ...
			</entity-mappings


		No caso acima, os nomes das colunas que foram alteradas nas anotações serão ignorados e os nomes padrões para os campos serão adotados (NAME e SALARY) em vez dos nomes definidos nas anotações;
