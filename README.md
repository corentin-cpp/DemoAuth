
# Demo Authentification

Cette application à été developpé dans un but éducatif. Aucune données sont enregistés !

Des fonctionnalités ont été supprimés et d'autres ajouter. Le projet est un projet ASP.NET Core MVC 6, j'ai ajouté un système de connexion de Compte Individuel

## Startup Project

- Créer un projet ASP.NET 6 MVC avec une authentification de compte individuel

### Ajouter les migrations 
- Ouvrez le fichier appsettings.json et modifier la Chaine de connection par défault par la chaine de connection souhaité afin de la relié a votre base de donnnée
- Dans la Console de Gestionnaire de package entrer la commande suivante `update-database`
- Une fois la création de la base de donnée réalisé vous pouvez aller vérifier que les tables ont correctement été ajouté

### Generation des pages Razor Identity
- Si besoins vous pouvez générer les pages de connexion pour se faire Cliquez Droit sur le dossier Data -> Ajouter -> Nouvel élément généré automatiquement (sur le menu à gauche) Selectionné Identité puis ajouter
- Selectionné Remplacer tous les fichiers
- Saississez Classe DbContext class ApplicationDbContect
- ajouter
## Modifications appliqués


- Modifications des pages de Connexions et Inscription
- Ajout de rôle adminsitrateur
- Gestions des rôles de l'applications
- Suppression d'envoie de mail de confirmation


## Ajout de rôle Adminstrateur & Standard


- Ajouter un controller RoleController
- Définissez RoleController
```csharp
        public RoleController(RoleManager<IdentityRole> roleManager) 
        {
            _roleManager = roleManager; 
        }
```

- La vue Index doit retourner la liste des rôles existant
```csharp
        public IActionResult Index()
        {
            var roles = _roleManager.Roles.ToList();

            return View(roles);
        }
```

- Ensuite Générer la vue Index et Coller ce code dans Role\Index.cshtml: 
```cshtml
@model IEnumerable<Microsoft.AspNetCore.Identity.IdentityRole>

<h1>List Of Roles</h1>
<a asp-action="CreateRole">Create</a>
<table class="table table-striped table-bordered">
    <thead>
        <tr>
            <td>Id</td>
            <td>Name</td>
            <td>||</td>
        </tr>
    </thead>
    <tbody>
        @foreach(var role in Model)
        {
            <tr>
                <td>@role.Id</td>
                <td>@role.Name</td>
                <td><a asp-action="DeleteRole">Delete</a></td>
            </tr>
        }
    </tbody>
</table>
```
- On va maintenant ajouter une vue permettant de créer un rôle
- Créer une action CreateRole de type HttpGet qui retourne un nouveau IdentityRole 
```csharp
        [HttpGet]
        public IActionResult CreateRole()
        {
            return View(new IdentityRole());
        }
```

- Ensuite on va créer la tâche qui permet de Creér le rôle dans notre base de données
```csharp
        [HttpPost]
        public async Task<IActionResult> CreateRole(IdentityRole role)
        {
            await _roleManager.CreateAsync(role);

            return View();
        }
```

- Générer la vue Create Role et copier ce code dans Role\Create.cshtml
```cshtml
@model Microsoft.AspNetCore.Identity.IdentityRole

@{
    ViewData["Title"] = "Create Role";
}

<h1>@ViewData["Title"]</h1>

<div class="row">
    <div class="col-md-4">
        <form method="post">
            <h4>Create new role</h4>
            <hr />
            <div asp-validation-summary="All" class="text-danger"></div>
            <div class="form-control">
                <label asp-for="Name"></label>
                <input asp-for="Name" class="form-control"/>
                <span asp-validation-for="Name" class="text-danger"/>
            </div>
            <button type="submit" class="btn btn-primary">Create</button>
        </form>
    </div>
</div>

@section Scripts{
    <partial name="_ValidationScriptsPartial"/>
}
```
- Ensuite vous pouvez ajouter les deux Rôles (Default et Admin).
- Si vous souhaitez attribuer ces rôles par défault lors de la création d'un compte il vous suffit de vous rendre dans Areas\Identity\Pages\Account\Register.cshtml.cs et d'insérer : 

`private readonly RoleManager<IdentityRole> _roleManager;`

puis ajouter 
`RoleManager<IdentityRole> roleManager` dans public RegisterModel et `_roleManager = roleManager;`

Ensuite dans la méthode OnPostAsync adapter le code selon vos besoins : 

```csharp
if (ModelState.IsValid)
            {
                var user = CreateUser();

                await _userStore.SetUserNameAsync(user, Input.Email, CancellationToken.None);
                await _emailStore.SetEmailAsync(user, Input.Email, CancellationToken.None);
                var result = await _userManager.CreateAsync(user, Input.Password);

                if (result.Succeeded)
                {
                    var defaultrole = _roleManager.FindByNameAsync("Default").Result;

                    _logger.LogInformation("L'utilisateur vient de créer un compte !");

                    var userId = await _userManager.GetUserIdAsync(user);
                    var code = await _userManager.GenerateEmailConfirmationTokenAsync(user);
                    code = WebEncoders.Base64UrlEncode(Encoding.UTF8.GetBytes(code));
                    var callbackUrl = Url.Page(
                        "/Account/ConfirmEmail",
                        pageHandler: null,
                        values: new { area = "Identity", userId = userId, code = code, returnUrl = returnUrl },
                        protocol: Request.Scheme);

                    await _emailSender.SendEmailAsync(Input.Email, "Confirm your email",
                        $"Please confirm your account by <a href='{HtmlEncoder.Default.Encode(callbackUrl)}'>clicking here</a>.");

                    if (defaultrole != null)
                    {
                        await _userManager.AddToRoleAsync(user, defaultrole.Name);
                    }

                    if (_userManager.Options.SignIn.RequireConfirmedAccount)
                    {
                        return RedirectToPage("RegisterConfirmation", new { email = Input.Email, returnUrl = returnUrl });
                    }
                    else
                    {
                        await _signInManager.SignInAsync(user, isPersistent: false);
                        return LocalRedirect(returnUrl);
                    }
                }
                foreach (var error in result.Errors)
                {
                    ModelState.AddModelError(string.Empty, error.Description);
                }
            }
```

Maintant lors de l'inscription d'un membre il aura par défault le rôle Défault

### Code Final du controller

```csharp
    [Authorize(Roles = "Admin")]
    public class RoleController : Controller
    {
        private RoleManager<IdentityRole> _roleManager;

        public RoleController(RoleManager<IdentityRole> roleManager) 
        {
            _roleManager = roleManager;
        }

        public IActionResult Index()
        {
            var roles = _roleManager.Roles.ToList();

            return View(roles);
        }

        [HttpGet]
        public IActionResult CreateRole()
        {
            return View(new IdentityRole());
        }

        [HttpPost]
        public async Task<IActionResult> CreateRole(IdentityRole role)
        {
            await _roleManager.CreateAsync(role);

            return View();
        }
    }
```

