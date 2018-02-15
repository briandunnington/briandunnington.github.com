Title: MockMock - A mocking framework for UWP
Date: 2015.08.31
Summary: ...or: how to generate classes on the fly when your language doesn't allow it

<!-- Main hero unit for a primary marketing message or call to action -->
<div class="hero-unit">
<h1>MockMock - A mocking framework for UWP</h1>
<p>...or: how to generate classes on the fly when your language doesn't allow it</p>
</div>

At my current employer, we have made it a point to focus on quality throughout the company. Nowhere is that more important than in the software that we build for our clients. 'Quality' can mean a lot of different things and there are a multitude of factors that go into writing quality software, both tangible and intangible. Before a line of code is written, the story writing, grooming, and estimating process helps set clear goals and expectations. Coding standards and conventions keep everybody on the same page. Continuous integration and a rigid QA process ensure consistent, reliable output. But one tool in our toolkit that we rely on heavily to ensure quality code is [Test Driven Development][TDD]. By letting the tests drive the code development, we ensure that we are solving the right problems and at the same time increasing developer confidence that code changes will not have unforseen impacts on other parts of the system. 

With such a large focus on TDD, a good unit testing process is essential, and one of the key components is a solid mocking framework. By mocking the ancillary components and focusing on the system under test, we ensure that our code is doing what is expected, and just as importantly, not doing anything that isn't expected. As part of the Windows and System Integration practices, we normally rely on the vast library of available .NET mocking libraries available from the community, such as [Moq][], [FakeItEasy][], and [JustMock][]. Writing mocks by hand can be both tedious and error-prone, but these tools make creating mocks quick and easy. Behind the scenes, nearly all .NET mocking libraries rely on the language's ability to dynamically produce IL code at runtime, specifically the classes in the `System.Reflection.Emit` namespace. By leveraging this powerful feature, the mocking libraries can reflect over classes or interfaces and dynamically construct a mock implementation that still satisfies the type-safety requirements of the language while simultaneously allowing for hooks to monitor and instrument the code execution.

However, with the advent of Windows 8 and <del>Metro</del>/<del>Modern</del>/<del>Windows Store</del>/Universal apps, .NET developers suddenly had the new WinRT-based framework to contend with. One of the changes was that the `System.Reflection.Emit` assemblies were no longer available. This essentially meant that an [entire class of mocking libraries no longer worked][goodbye]. There were a few attempts at workarounds, but none of them provided the simplicity and ease-of-use of the standard tools. Any barrier to quickly and easily writing and running tests undermines the effectiveness of TDD, so we knew that we would need to come up with a solution.

I approached the problem from several vantage points and came up with a few ideas. I tried leveraging the [dynamic capabilities of the language][dynamic] to create code on the fly, but the language's type safety features would not allow an object to be both dynamic and a valid interface implementation at the same time. I tried creating our unit test projects targeted at the full .NET framework rather than the WinRT subset, and although that allowed us to reference the existing mocking tools, the new testing framework would not allow non-WinRT-based test projects to target a WinRT-based application.

The approach I finally settled on was to leverage the [Text Template Transformation Toolkit (T4)][T4] functionality. T4 essentially lets you create code templates and then execute actual .NET code to generate new output. In our case, I used C# code to generate more C# code as the output, which could then be consumed by our unit test projects. I wrote some code that scanned a list of assemblies for any interfaces, and then reflected over the interface members to determine which fields, properties, methods, and events needed to be implemented. Since we had previously been fond of using Telerik's JustMock Lite library and its [Arrange/Act/Assert][AAA] pattern, I decided to output API-compatible replacement mocks. So instead of referencing JustMock Lite and letting it dynamically generate the mocks on the fly, the T4 template was set up to automatically create the mocks at build time which could then be used in the unit tests in the same Arrange/Act/Assert pattern with the same API that developers were familiar with.

#### T4 template
	<#@ template debug="false" hostspecific="false" language="C#" #>
	<#@ assembly name="System.Core" #>
	<#@ assembly name="System.Runtime" #>
	<#@ import namespace="System" #>
	<#@ import namespace="System.IO" #>
	<#@ import namespace="System.Text" #>
	<#@ import namespace="System.Reflection" #>
	<#@ import namespace="System.Linq" #>
	<#@ import namespace="System.Collections.Generic" #>
	<#@ output extension=".cs" #>

	using System;

	namespace Tests.Mocks
	{
	<#
		var formatTypeName = new Func<Type, string>((type) => {
			var typeName = type.ToString();
			typeName = typeName.Replace("System.Void", "void");
			typeName = typeName.Replace("`1", "");
			typeName = typeName.Replace("`2", "");
			typeName = typeName.Replace("`3", "");
			typeName = typeName.Replace("[", "<");
			typeName = typeName.Replace("]", ">");
			if(typeName.EndsWith("&")) typeName = "out " + typeName.Replace("&", "");
			return typeName;
		});

        var anyTypeFromAssembly = typeof(App.Sample);
        var namespaceBuilder = new StringBuilder();
        var assembly = Assembly.GetAssembly(anyTypeFromAssembly);
        var types = assembly.GetTypes();
        foreach(var type in types)
        {
            if(type.IsInterface)
            {
                var properties = new List<PropertyInfo>();
                var methods = new List<MethodInfo>();
                var events = new List<EventInfo>();
                var interfaces = new List<Type>();
                interfaces.Add(type);
                interfaces.AddRange(type.GetInterfaces());
                foreach(var i in interfaces)
                {
                    properties.AddRange(i.GetProperties());
                    methods.AddRange(i.GetMethods());
                    events.AddRange(i.GetEvents());
                }
	#>
	public class <#= "Mock" + formatTypeName(type).Replace("global::", "").Replace(type.Namespace, "").Substring(2) #> : MockBase<<#= formatTypeName(type) #>>, <#= formatTypeName(type) #>
	{
	<#
                foreach(var property in properties)
                {
	#>
		private <#= formatTypeName(property.PropertyType) #> _<#= property.Name #>;
		public <#= formatTypeName(property.PropertyType) #> <#= property.Name #> 
		{
			get
			{
				try
				{
					throw new StackTraceHelperForMockingException();
				}
				catch(Exception exx)
				{
					<#= formatTypeName(property.PropertyType) #> val;
					RecordProperty<<#= formatTypeName(property.PropertyType) #>>(exx, _<#= property.Name #>, out val);
					return val;
				}
			}
	<#				if(property.CanWrite)
				{
	#>
			set
			{
				_<#= property.Name #> = value;
			}
	<#
					}
	#>
		}
	<#
				}
                foreach(var method in methods)
                {
					if(method.Name.StartsWith("get_") || method.Name.StartsWith("set_") || method.Name.StartsWith("remove_") || method.Name.StartsWith("add_")) continue;
                    var parameterList = "";
                    var anonymousObjectMemberList = "(object)null";
					var outParameterSetters = "";
                    var parameters = method.GetParameters();
                    if(parameters != null && parameters.Length > 0)
                    {
                        parameterList = String.Join(", ", parameters.Select((p) => formatTypeName(p.ParameterType) + " " + p.Name));
                        anonymousObjectMemberList = "new { " + String.Join(", ", parameters.Select((p) => p.Name)) + " }";
						outParameterSetters = String.Join(" ", parameters.Where((p) => p.IsOut).Select((p) => p.Name + " = default(" + p.ParameterType.ToString().Replace("&", "") + ");"));
                    }
					var hasReturnValue = method.ReturnType.Name == "Void" ? false : true;
					var returnType = hasReturnValue ? "<" + formatTypeName(method.ReturnType) + ">" : "";
					var methodName = method.Name;
					var genericArguments = method.GetGenericArguments();
					if(genericArguments != null && genericArguments.Length > 0)
					{
						methodName = methodName + "<" + String.Join(", ", genericArguments.Select((g) => g.Name)) + ">";
					}
	#>
		public <#= formatTypeName(method.ReturnType) #> <#= methodName #>(<#= parameterList #>)
		{   <#= outParameterSetters #>
			try
			{
				throw new StackTraceHelperForMockingException();
			}
			catch(Exception exx)
			{
				var obj = <#= anonymousObjectMemberList #>;
	<#
					if(hasReturnValue)
					{
	#>
				<#= formatTypeName(method.ReturnType) #> val;
				var called = Record<#= returnType #>(exx, obj, out val);
				return val;
	<#
					}
					else
					{
	#>
				var called = Record(exx, obj);
	<#
					}
	#>
			}
		}
	<#
                }
				foreach(var ev in events)
				{
	#>
		public event <#= formatTypeName(ev.EventHandlerType) #> <#= ev.Name #>;
	<#
				}
	#>
	}
	<#
            }
        }
	#>
	}

#### Example Unit Test
        public async Task SignUp_returns_UserAuth_for_valid_info()
        {
            // arrange
            string expectedEmail = "test@test.email";
            string expectedUsername = "testusername";
            string expectedPassword = "testpassword";
            var expectedAuthToken = "ABC!@#123";
            var expectedUserId = 1234;
            var expectedUserAuth = new UserAuth() { Auth = expectedAuthToken, User = new User() { UserId = expectedUserId } };
            var mockSessionManager = Mock.Create<ISessionManager>();
            var mockSession = Mock.Create<ISession>();
            Mock.Arrange(() => mockSession.Signup(expectedEmail, expectedPassword, expectedUsername, Arg.AnyString)).Returns(Task.FromResult(expectedUserAuth));
            Mock.Arrange(() => mockSessionManager.GetSession()).Returns(Task.FromResult(mockSession));

            // act
            var userManager = new UserManager(mockSessionManager);
            var user = await userManager.SignUp(expectedEmail, expectedPassword, expectedUsername, null);

            // assert
            Assert.IsNotNull(user);
            Assert.AreEqual(expectedUserId, user.UserId);
            mockSessionManager.Assert();
            mockSession.Assert();
        }

When the prerelease bits of Windows 10 and the Universal Windows Platforms (UWP) apps came out, the restriction on `System.Reflection.Emit` still existed but the custom T4 solution still worked great. By the time the developer tools were RTMed, `System.Reflection.Emit` had been restored to the framework and was suddenly available again. The system had been working so good for us that I was able to simply replace the T4 code generation with dynamic proxy generation behind the scenes and all of our unit tests continued to work unchanged. In the end, it was great to see Microsoft resurrect the functionality but it was even better knowing that we had solved the problem and come up with an elegant solution to ensure that we continued to be able to produce maintainable, testable, quality software.

PS: The updated code (no longer uses the T4 template technique) is available here: [MockMock][]

[TDD]: https://en.wikipedia.org/wiki/Test-driven_development
[Moq]: https://github.com/moq/moq4
[FakeItEasy]: https://fakeiteasy.github.io/
[JustMock]: https://github.com/telerik/JustMockLite
[goodbye]: http://geekswithblogs.net/mbrit/archive/2012/06/05/say-goodbye-to-system.reflection.emit-any-dynamic-proxy-generation-in-winrt.aspx
[dynamic]: https://msdn.microsoft.com/en-us/library/dd264736.aspx
[T4]: https://msdn.microsoft.com/en-us/library/bb126445.aspx
[AAA]: http://c2.com/cgi/wiki?
[MockMock]: https://github.com/briandunnington/MockMock
