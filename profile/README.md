# Overmock
![DOTNET Build](https://github.com/overmock/overmock/actions/workflows/dotnet.yml/badge.svg)

Overmock is a mocking framework in development that allows for creating dynamic proxies that monitor and control expected behavior when writing unit tests. Here are some examples below.

The current goal is optimizing the proxy member delegate compilation in to a few different implementations using DynamicMethod and Expressions. Expressions for methods is currently working and in the latest release. Compiling delegates for properties is next then fixing the IL generation for the DynamicMethod delegates. 

``` C#
[TestClass]
public class ExampleTestsForReadMe
{
    public class Model
    {
        public int Id { get; set; }
    }
    public interface IRepository
    {
        bool Save(Model model);
    }
    public interface ILog
    {
        void Log(string message);
    }
    public class Service
    {
        private readonly ILog _log;
        private readonly IRepository _repo;
        public Service(ILog log, IRepository repo)
        {
            _log = log;
            _repo = repo;
        }
        public void SaveModel(Model model)
        {
            try
            {
                var saved = _repo.Save(model);
                if (!saved)
                {
                    _log.Log("Failed to save");
                }
            }
            catch (Exception ex)
            {
                _log.Log(ex.Message);
                throw;
            }
        }
    }

    [TestMethod]
    public void CallsSaveTest()
    {
        var id = 22;
        var wasSaved = false;
        var log = Overmocked.ExpectAnyInvocation<ILog>();
        var repository = Overmocked.Interface<IRepository>();

        repository.Override(r => r.Save(Its.Any<Model>())).ToCall(c =>
        {
            wasSaved = true;
            return c.Get<Model>("model")?.Id == id;
        }, Times.Once);

        var service = new Service(log.Target, repository.Target);
        service.SaveModel(new Model { Id = id });

        Assert.IsTrue(wasSaved);
    }

    [TestMethod]
    public void ThrowsExceptionWhenSaveFailsTest()
    {
        var expected = "Failed to save";
        var log = Overmocked.ExpectAnyInvocation<ILog>();
        var repository = Overmocked.Interface<IRepository>();

        repository.Override(r => r.Save(Its.Any<Model>())).ToThrow(new Exception(expected));

        var service = new Service(log.Target, repository.Target);

        try
        {
            service.SaveModel(new Model());

            Assert.Fail("SaveModel Failed to throw an exception.");
        }
        catch (Exception actual)
        {
            Assert.AreEqual(expected, actual.Message);
        }
    }
}
```
