```
using System.Data;

//資料庫撈取的資料
var menuData = new List<MenuTableModel> {
    new MenuTableModel { ID = "1" , Name = "1" },
    new MenuTableModel { ID = "1-1" , Name = "1-1",ParentID = "1" },
    new MenuTableModel { ID = "1-2" , Name = "1-2",ParentID = "1" },
    new MenuTableModel { ID = "1-3" , Name = "1-3",ParentID = "1" },
    new MenuTableModel { ID = "1-2-1" , Name = "1-2-1",ParentID = "1-2" },
    new MenuTableModel { ID = "1-2-2" , Name = "1-2-2",ParentID = "1-2" },
    new MenuTableModel { ID = "2" , Name = "2" },
    new MenuTableModel { ID = "2-1" , Name = "2-1",ParentID = "2" },
    new MenuTableModel { ID = "2-2" , Name = "2-2",ParentID = "2" },
    new MenuTableModel { ID = "2-3" , Name = "2-3",ParentID = "2" },
    new MenuTableModel { ID = "2-2-1" , Name = "2-2-1",ParentID = "2-2" },
    new MenuTableModel { ID = "2-2-2" , Name = "2-2-2",ParentID = "2-2" },
};

//取得第0層的子Menu
var result = GetChildMenu(menuData, string.Empty);

Console.Read();

//遞迴取得子Menu，思考方向針對需要遞迴的部分抽成一個方法
List<Menu> GetChildMenu(List<MenuTableModel> datatSource, string parentID)
{
    var dataSouceChildMenu = datatSource.Where(m => m.ParentID == parentID).ToList();

    var resultChildMenu = new List<Menu>();

    if (dataSouceChildMenu.Any())
    {
        foreach (var item in dataSouceChildMenu)
        {
            var menu = new Menu();
            menu.ID = item.ID;
            menu.Name = item.Name;
            menu.ParentID = item.ParentID;
            menu.ChildMenu = GetChildMenu(datatSource, menu.ID);
            resultChildMenu.Add(menu);
        }
    }

    return resultChildMenu;
}


// Table Model
public class MenuTableModel
{
    public string ID { get; set; }
    public string Name { get; set; }
    public string ParentID { get; set; } = string.Empty;
}

// ViewModel
public class Menu
{
    public string ID { get; set; }
    public string Name { get; set; }

    public string ParentID { get; set; } = string.Empty;

    public List<Menu> ChildMenu { get; set; } = new List<Menu>();

}
```
