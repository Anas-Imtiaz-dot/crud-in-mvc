#Crud in MVC Using  Ajax
#Model Class
public class Employees
{
    [Key]
    public int EmpId { get; set; }
    public string FullName { get; set; } = string.Empty;
    public string Contact { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Address { get; set; } = string.Empty;

  public string ImagePath {  get; set; } = string.Empty;

    [NotMapped]
    public  IFormFile? Image {  get; set; }
    public int DepartmentDid { get; set; }

    // Navigation property for the related Department
    [ForeignKey("Departments")]
    public virtual Departments? Department { get; set; }
}
#Employee Controller 
    public class EmployeesController : Controller
    {
        private EFCoreDbContext context;

        public EmployeesController(EFCoreDbContext _context)
        {
            context = _context;
        }


        public IActionResult Index()
        {
            return View();
        }
        [HttpGet]
        public PartialViewResult EmployeeListAndForm()
        {
            try
            {
                var departments = context.Departments.ToList();
                ViewBag.Departments = departments;
                return PartialView("EmployeeListAndForm");
            }

            catch (Exception ex)
            {
                // Handle exception (e.g., log it)
                return PartialView("Error", ex.Message);
            }
        }
        public IActionResult GetEmployeList()
        {
            try
            {
                var employeeList = from e in context.Employees
                                   join d in context.Departments
                                   on e.DepartmentDid equals d.Did
                                   select new
                                   {
                                       empid = e.EmpId,
                                       fullname = e.FullName,
                                       contact = e.Contact,
                                       email = e.Email,
                                       address = e.Address,
                                       DepartmentName = d.Name,
                                      imagepath=e.ImagePath,
                                   };

                var list = employeeList.ToList(); // Execute the query and retrieve the results

                return Json(new JsonRespons { IsSuccess = true, Data = list });
            }
            catch (Exception ex)
            {
                var msg = ex.InnerException != null ? ex.InnerException.Message : ex.Message;
                return Json(new JsonRespons { IsSuccess = false, Data = msg });
            }
        }



        [HttpPost]
        public IActionResult SaveEmployees(Employees model , IFormFile Image)
        {
            try
            {
                bool contactExists = context.Employees
                                             .Any(e => e.Contact == model.Contact);

                bool EmailExist = context.Employees
                    .Any(e => e.Email == model.Email);



                if (contactExists)
                {
                    return Json(new JsonRespons() { IsSuccess = false, Message = "An employee with the same contact already exists." });
                }

                else if (EmailExist)
                {
                    return Json(new JsonRespons() { IsSuccess = false, Message = "An employee with the same email already exists." });
                }

                if (Image != null && Image.Length > 0)
                {
                    var uploadsFolder = Path.Combine(Directory.GetCurrentDirectory(), "wwwroot/picture/Employee-Picture");
                    var filePath = Path.Combine(uploadsFolder, Image.FileName);

                    using (var stream = new FileStream(filePath, FileMode.Create))
                    {
                        Image.CopyTo(stream);
                    }

                    // Set the relative path to be saved in the database
                    model.ImagePath = "/picture/Employee-Picture/" + Image.FileName;
                }


                if (model.EmpId > 0)
                {

                    context.Employees.Update(model);
                }
                else
                {
                    context.Employees.Add(model);


                }

                context.SaveChanges();
                return Json(new JsonRespons() { IsSuccess = true, Message = "Record saved successfully.", Data = model });
            }
            catch (Exception ex)
            {
                var msg = ex.InnerException != null ? ex.InnerException.Message : ex.Message;
                return Json(new JsonRespons() { IsSuccess = false, Message = msg });
            }
        }



        [HttpGet]
        public IActionResult Edit(int empid)
        {
            // Ensure correct table and data are being used
            try
            {
                var obj = context.Employees.Find(empid);
                if (obj == null)
                {
                    return Json(new JsonRespons { IsSuccess = false, Message = "Employees not found" });
                }
                return Json(new JsonRespons { IsSuccess = true, Message = "Record Load to form  sucessfully", Data = obj });
            }
            catch (Exception ex)
            {
                var msg = ex.InnerException != null ? ex.InnerException.Message : ex.Message;
                return Json(new JsonRespons { IsSuccess = false, Message = msg });

            }
        }

        public IActionResult SearchEmployees(string Search, int departmentId)
        {
            var filteredList = from e in context.Employees
                               join d in context.Departments on e.DepartmentDid equals d.Did
                               select new
                               {
                                   empid = e.EmpId,
                                   fullname = e.FullName,
                                   contact = e.Contact,
                                   email = e.Email,
                                   address = e.Address,
                                   imagepath=e.ImagePath,
                                   DepartmentName = d.Name,
                                   e.DepartmentDid
                               };
            if (!string.IsNullOrEmpty(Search) && departmentId > 0)
            {
                filteredList = filteredList.Where(e =>
                    (e.fullname.Contains(Search) || e.email.Contains(Search)) && e.DepartmentDid == departmentId);
            }
            else if (!string.IsNullOrEmpty(Search))
            {
                filteredList = filteredList.Where(e => e.fullname.Contains(Search) || e.email.Contains(Search));
            }
            else if (departmentId > 0)
            {
                filteredList = filteredList.Where(e => e.DepartmentDid == departmentId);
            }

            var result = filteredList.ToList();

            return Json(new JsonRespons() { IsSuccess = true, Data = result });
        }

        public IActionResult Details(int empid)
        {
            var employee = context.Employees
                                  .FirstOrDefault(e => e.EmpId == empid);

            if (employee != null)
            {
                // Fetch the department name
                var departmentName = context.Departments
                                            .Where(d => d.Did == employee.DepartmentDid)
                                            .Select(d => d.Name)
                                            .FirstOrDefault();

                // Use ViewBag to pass the department name to the view
                ViewBag.DepartmentName = departmentName;

                return View(employee);
            }

            return View(); // Optionally, you could return a NotFound or similar response.
        }

    }
    #Indexof Employee 
    
@{
    ViewData["Title"] = "Index";
}

<div class="text-center bg-light py-3 text-white shadow-sm border">
    <h1 class="fw-bold text-dark">Employees</h1>
    <a id="LoadDepartments" class="btn btn-info">
        <i class="fa-solid fa-refresh"></i> Load Employees
    </a>
    <a id="SetTemplate" class="btn btn-warning">
        <i class="fa-solid fa-file-edit"></i> Set Template
    </a>
    <a class="btn btn-secondary me-1">
        <i class="fa-solid fa-download"></i> Download Template
    </a>
    <a id="SetTemplateRules" class="btn btn-success me-1">
        <i class="fa-solid fa-pencil"></i> Update Excel Rules
    </a>
    <a id="uploadBulkData" class="btn btn-danger me-1">
        <i class="fa-solid fa-upload"></i> Upload Bulk Data
    </a>
</div>

<div id="contentDiv">

    @await Html.PartialAsync("~/Views/Shared/EmployeeListAndForm.cshtml")

</div>


<script>
    $(document).ready(function () {
        LoadEmployeesFormAndList()
    });
    function LoadEmployeesFormAndList() {
        $.ajax({
            type: 'GET',
            url: '@Url.Action("EmployeeListAndForm", "Employees")',


            success: function (response) {
                $('#contentDiv').html(response);
            },
            error: function (err) {
                console.log(err);
                alert('Ajax request call nhi horahi  ' + err.responseText);
            }



        });
    }
</script>
#GetEmployeeFormAndList Partial view


@* @model IEnumerable<WebApplication1.Models.Departments> *@



      <script src="~/js/site.js" asp-append-version="true"></script>
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css">
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>

    <style>
        .empimage {
            width: 50px;
            height: 50px;
            overflow:hidden;
        }
    </style>


     <div class="container-fluid">
    <form id="Employees-Form" method="post" enctype="multipart/form-data" asp-controller="Employees" asp-action="SaveEmployees">
        <a id="Create-New" class="btn btn-primary btn-sm">
            <i class="fa-solid fa-plus-circle"></i>
            Create New
        </a>
        <br />
        <div class="form-group">
            <label for="Photo">Photo</label>
            <input type="file" id="Photo" name="Image" class="form-control" />
            <span class="text-danger" id="PhotoValidation"></span>
        </div>
            <div class="form-group">
                <label for="FullName">Full Name</label>
                <input id="FullName" name="FullName" class="form-control" />
                <span class="text-danger" id="FullNameValidation"></span>
            </div>

            <div class="form-group">
                <label for="Contact">Contact</label>
                <input id="Contact" name="Contact" class="form-control" />
                <span class="text-danger" id="ContactValidation"></span>
            </div>

            <div class="form-group">
                <label for="Email">Email</label>
                <input id="Email" name="Email" class="form-control" />
                <span class="text-danger" id="EmailValidation"></span>
            </div>

            <div class="form-group">
                <label for="Address">Address</label>
                <input id="Address" name="Address" class="form-control" />
                <span class="text-danger" id="AddressValidation"></span>
            </div>
        <div class="form-group">
            <label for="DepartmentDid" class="form-label">Department</label>
            <select id="departmentdid" name="departmentdid" class="form-control">
                <option value="">Choose</option>
                @if (ViewBag.Departments != null)
                {
                    var departments = ViewBag.Departments as List<WebApplication1.Models.Departments>;
                    if (departments != null)
                    {
                        foreach (var department in departments)
                        {
                            <option value="@department.Did">@department.Name</option>
                        }
                    }
                    else
                    {
                        <option value="">No departments available</option>
                    }
                }
                else
                {
                    <option value="">Departments not loaded</option>
                }
            </select>
        </div>
            

            <button type="button" class="btn btn-success" id="btnsave">Save</button>
        </form>
        <br />
        <div class="row">
            <div class="form-horizontal">
                <div class="form-group">
                    <label class="col-md-6 control-label"><b>Search</b></label>
                    <div class="col-md-6">
                        <input type="text" id="searchInput" name="Search" class="form-control" placeholder="Search your text">
                    </div>
                    <br />
                    <h5>Search for Employees</h5>
                    <div>

                    <label for="DepartmentDid" class="form-label">Department</label>
                    <input type="text" id="changedDepartment" class="noshow" value="Mickey" hidden />
                    <select id="DepartmentDid" onchange=" FilterEmployeesByDepartment(this.value)" name="DepartmentDid" class="form-control">
                        
                        <option value="">Choose</option>
                        @if (ViewBag.Departments != null)
                        {
                            var departments = ViewBag.Departments as List<WebApplication1.Models.Departments>;
                            if (departments != null)
                            {
                                foreach (var department in departments)
                                {
                                    <option value="@department.Did">@department.Name</option>
                                }
                            }
                            else
                            {
                                <option value="">No departments available</option>
                            }
                        }
                        else
                        {
                            <option value="">Departments not loaded</option>
                        }
                    </select>
                        
                    <br />
                    </div>
                    <div class="col-md-2">
                        <button id="searchButton" class="btn btn-primary" type="button">Search</button>
                    </div>
                </div>
            </div>
            <div class="col-md-2">
                <button id="backToListButton" class="btn btn-secondary" type="button">Back to List</button>
            </div>
        </div>
        <br />

        <h3 class="text-primary">Employee List</h3>
        <table class="table table-bordered table-striped table-hover">
            <thead>
                <tr>
                    <th>Employee Id</th>
                     
                    <th>Full Name</th>
                    <th>Email</th>
                    <th>Contact</th>
                    <th>Address</th>
                    <th>Department</th>
                     <th>Photo</th>
                    <th>Action</th>
                    
                </tr>
            </thead>
            <tbody id="employeeListDiv">
                <!-- Dynamically populate rows using JavaScript -->
            </tbody>
        </table>
    </div>

<script>
    $(document).ready(function () {
       
        GellAllEmployees();

        $("#searchInput").on("input", function () {
            var searchTerm = $(this).val(); // Get the search term from the input field
            var departmentId = document.getElementById('changedDepartment').value; // Get the selected department ID
            console.log('DepartmentChanged: ', departmentId);
            SearchEmployees(searchTerm, departmentId); // Call the function to search employees
        });
        // Handle back to list button click
        $('#backToListButton').click(function () {
            console.log('Back to List button clicked');
            GellAllEmployees(); // Reload the full employee list
        });

        // Handle create new button click
        $('#Create-New').click(function () {
            $('#Employees-Form')[0].reset(); // Reset the form
            $('#Employees-Form').find('input').val('');
        });

        // Handle department dropdown change
        $("#DepartmentDid").on("change", function () {
            var selectedDepartmentId = $(this).val(); // Get the selected department ID
            var searchTerm = $('#searchInput').val(); // Get the current search term
            SearchEmployees(searchTerm, selectedDepartmentId); // Call the function to filter employees
        });

    });

    //search Department by drop down
    function FilterEmployeesByDepartment(departmentId) {
        document.getElementById('changedDepartment').value = departmentId;
        console.log('Department ID NOW: ', departmentId);
            $.ajax({
                type: 'GET',
            url: '@Url.Action("SearchEmployees", "Employees")',
                data: {departmentId: departmentId },
                success: function (response) {
                    if (response.isSuccess) {
                        var list = response.data;
                        var row = '';

                        for (var i = 0; i < list.length; i++) {
                            // console.log(list);
                            row += `<tr data-did="${list[i].did}">
                                   <td>${list[i].empid}</td>
                                    <td>${list[i].fullname}</td>
                                    <td>${list[i].email}</td>
                                    <td>${list[i].contact}</td>
                                    <td>${list[i].address}</td>
                                    <td>${list[i].departmentName}</td>
                                  

                                    <td>
                                        <button type="button" class="btn btn-warning" onclick="Edit(${list[i].empid},this)"><i class="fa-solid fa-edit"></i></button> |
                                        <button type="button" class="btn btn-danger" onclick="Delete(${list[i].empid},this)"><i class="fa-solid fa-trash"></i></button>
                                    </td>
                                </tr>`;
                        }
                        $("#employeeListDiv").html(row);
                    } else {
                        $("#employeeListDiv").html("<tr><td colspan='7'>Error: " + response.message + "</td></tr>");
                    }
                },
                error: function (err) {
                    alert('An error occurred while processing your request.');
                    $("#employeeListDiv").html("<tr><td colspan='7'>An error occurred while processing your request.</td></tr>");
                }
            });
        }


        //Search Employees 
        function SearchEmployees(searchTerm,departmentId) {

            $.ajax({
                type: 'GET',
                url: '@Url.Action("SearchEmployees", "Employees")', // Adjust controller name if necessary
            
            data: { Search: searchTerm, departmentId: departmentId },
               
              
                success: function (response) {
                    if (response.isSuccess) {
                        var list = response.data;
                        var row = '';

                        for (var i = 0; i < list.length; i++) {
                            row += `<tr data-did="${list[i].did}">
                                                <td>${list[i].empid}</td>
                                               <td>${list[i].fullname}</td>
                                              <td>${list[i].email}</td>
                                                <td>${list[i].contact}</td>
                                                <td>${list[i].address}</td>
                                               <td>${list[i].departmentName}</td>
                                             
                                    <td>
                                            <button type="button" class="btn btn-warning" onclick="Edit(${list[i].empid},this)"><i class="fa-solid fa-edit"></i></button> |
                                        <button type="button" class="btn btn-danger" onclick="Delete(${list[i].empid},this)"><i class="fa-solid fa-trash"></i></button>
                                    </td>
                                    </tr>`;
                        }
                        $("#employeeListDiv").html(row);
                    } else {
                        $("#employeeListDiv").html("<tr><td colspan='3'>Error: " + response.message + "</td></tr>");
                    }
                },
                error: function (err) {
                    alert('An error occurred while processing your request.');
                    $("#employeeListDiv").html("<tr><td colspan='3'>An error occurred while processing your request.</td></tr>");
                }
            });
        }

        function GellAllEmployees() {
        $.ajax({
            
                type: 'GET',
            url: '@Url.Action("GetEmployeList", "Employees")',
                success: function (response) {

                    if (response.isSuccess) {
                        var html = '';
                        var list = response.data;
                        for (var i = 0; i < list.length; i++) {
                          
                            html += `<tr>
                                    <td>${list[i].empid}</td>
                                    <td>${list[i].fullname}</td>
                                    <td>${list[i].email}</td>
                                    <td>${list[i].contact}</td>
                                     <td>${list[i].address}</td>
                                      <td>${list[i].departmentName}</td>
                                            <td><img class="empimage" src="${list[i].imagepath}" alt="Employee Image" /></td>
                                            <td>
                                        <button type="button" class="btn btn-warning" onclick="Edit(${list[i].empid},this)"><i class="fa-solid fa-edit"></i></button> |
                                    <button type="button" class="btn btn-danger" onclick="Delete(${list[i].empid},this)"><i class="fa-solid fa-trash"></i></button>|
                                       <button type="button" class="btn btn-info" onclick="Details(${list[i].empid},this)"> <i class="fa-solid fa-info-circle"></i> Details </button>

                                </td>
                            </tr>`;
                        }
                        $('#employeeListDiv').html(html);
                    } else {
                        alert('Error loading data:aaaaa ' + response.message);
                    }
                },
                error: function (err) {
                    notyf.open({ type: 'error', message: err.statusText });
                }
            });
        }

        $('#btnsave').click(function () {
        event.preventDefault(); // Prevent the default form submission
            var _empID = $('#EmpId').val(); // Ensure the ID field is correct
            console.log($('#Employees-Form').serialize());
     

        var formData = new FormData($('#Employees-Form')[0]);

            $.ajax({
                type: 'POST',
                url: '@Url.Action("SaveEmployees", "Employees")',
               data: formData,
               contentType: false, // Set this to false for file uploads
                processData: false,
                success: function (response) {
                    if (response.isSuccess) {
                        var data = response.data;
                        var row = `<tr>
                                <td>${data.empid}</td>
                                <td>${data.fullname}</td>
                                <td>${data.email}</td>
                                <td>${data.contact}</td>
                                <td>${data.address}</td>
                                <td>${data.departmentName}</td>
                                   <td><img class="empimage" src="${data.imagepath}" alt="Employee Image" /></td>
                            <td>
                                    <button type="button" class="btn btn-warning" onclick="Edit(${data.empid},this)"><i class="fa-solid fa-edit"></i></button> |
                                <button type="button" class="btn btn-danger" onclick="Delete(${data.empid},this)"><i class="fa-solid fa-trash"></i></button>
                            </td>
                        </tr>`;

                        if (_empID !== '') {
                            // Edit
                            if (currentRow) {
                                currentRow.replaceWith(row);
                            }
                        } else {
                            // Add new
                            $('#employeeListDiv').append(row);
                        }
                        notyf.open({ type: 'success', message: response.message || "Record saved successfully" });
                    } else {
                        notyf.open({ type: 'error', message: response.message || "Record cannot be saved" });
                    }
                },
                error: function (err) {
                    console.log(err.statusText);
                    notyf.open({ type: 'error', message: err.statusText });
                }
            });
        });


        function Edit(id, element) {
            console.log("Edit function called with id: " + id);
            currentRow = $(element).closest('tr');

            // Load the selected employee data into the form
            $.ajax({
                type: 'GET',
                url: '@Url.Action("Edit", "Employees")', // Ensure this URL is correct
                data: { empid: id }, // Ensure this parameter matches your action method's parameter
                success: function (response) {
                    console.log(response);
                    if (response.isSuccess) {
                        var data = response.data;
                        $('#EmpId').val(data.empid); // Ensure these IDs match your form input IDs
                        $('#FullName').val(data.fullname);
                        $('#Email').val(data.email);
                        $('#Contact').val(data.contact);
                        $('#Address').val(data.address);
                        $('#Name').val(data.departmentDid); // Assuming this is the correct field
                        $('#ImagePath').val(data.imagepath);
                        $('#Image').val(data.image);

                   
                    } 
                    else {
                        notyf.open({ type: 'error', message: response.message || "Failed to load employee data" });
                    }
                },
                error: function (err) {
                    notyf.open({ type: 'error', message: err.statusText });
                }
            });
        }
    function Details(empid) {
        window.location.href = '/Employees/Details?empid=' + empid;
    }
        // Define Delete function if required
        function Delete(id, element) {
            // Your delete code here
        }
   

</script>



    


