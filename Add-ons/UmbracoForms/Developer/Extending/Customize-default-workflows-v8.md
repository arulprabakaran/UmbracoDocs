---
versionFrom: 8.0.0
meta.Title: "Customize default workflows"
meta.Description: "How to amend the built-in behavior of associating workflows with new forms"
---

# Customize Default Workflows For a Form

The default behavior when a new form is created is for a single workflow to be added, which will send a copy of the form to the current backoffice user's email address.

From versions 8.13 it's possible to amend this behavior and change it to fit your needs.

## Implementing a Custom Behavior

The interface `IApplyDefaultWorkflowsBehavior` is used to abstract the logic for setting default workflows for a form.  The default behavior is defined using a built-in, internal class that implements this interface.

You can create your own implementation of this interface. An illustrative example of this, adding a custom workflow that writes to the log, is shown below.

Firstly, the custom workflow:

```c#
using System;
using System.Collections.Generic;
using Umbraco.Core.Composing;
using Umbraco.Core.Logging;
using Umbraco.Forms.Core.Attributes;
using Umbraco.Forms.Core.Enums;
using Umbraco.Forms.Core.Persistence.Dtos;

namespace MyNamespace
{
    public class TestWorkflow : WorkflowType
    {
        public const string TestWorkflowId = "7ca500a7-cb34-4a82-8ae9-2acac777382d";

        public TestWorkflow()
        {
            Id = new Guid(TestWorkflowId);
            Name = "Test Workflow";
            Description = "A test workflow that writes a log line";
            Icon = "icon-edit";
        }

        [Setting("Message", Description = "The log message to write", View = "TextField")]
        public string Message { get; set; }

        public override List<Exception> ValidateSettings()
        {
            var exs = new List<Exception>();
            if (string.IsNullOrEmpty(Message))
            {
                exs.Add(new Exception("'Message' setting has not been set"));
            }

            return exs;
        }

        public override WorkflowExecutionStatus Execute(Record record, RecordEventArgs e)
        {
            Current.Logger.Info<TestWorkflow>($"'{Message}' written at {DateTime.Now}"); ;
            return WorkflowExecutionStatus.Completed;
        }
    }
}
```

Secondly, the custom implementation of `IApplyDefaultWorkflowsBehavior`:

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using Umbraco.Core.IO;
using Umbraco.Forms.Core;
using Umbraco.Forms.Core.Enums;
using Umbraco.Forms.Core.Providers;
using Umbraco.Forms.Web.Behaviors;
using Umbraco.Forms.Web.Models.Backoffice;

namespace MyNamespace
{
    public class CustomApplyDefaultWorkflowsBehavior : IApplyDefaultWorkflowsBehavior
    {
        private readonly WorkflowCollection _workflowCollection;
        private readonly IFacadeConfiguration _facadeConfiguration;

        public CustomApplyDefaultWorkflowsBehavior(
            WorkflowCollection workflowCollection,
            IFacadeConfiguration facadeConfiguration)
        {
            _workflowCollection = workflowCollection;
            _facadeConfiguration = facadeConfiguration;
        }


        public void ApplyDefaultWorkflows(FormDesign form)
        {
            // If default workflow addition is disabled in configuration, then exit.
            if (_facadeConfiguration.GetSetting("DisableDefaultWorkflow").ToLower() == "true")
            {
                return;
            }

            // Retrieve the type of the default workflow to add.
            WorkflowType testWorkflowType = _workflowCollection[new Guid(TestWorkflow.TestWorkflowId)];

            // Create a workflow object based on the workflow type.
            var defaultWorkflow = new FormWorkflowWithTypeSettings
            {
                Id = Guid.Empty,
                Name = "Log a message",
                Active = true,
                IncludeSensitiveData = IncludeSensitiveData.False,
                SortOrder = 1,
                WorkflowTypeId = testWorkflowType.Id,
                WorkflowTypeName = testWorkflowType.Name,
                WorkflowTypeDescription = testWorkflowType.Description,
                WorkflowTypeGroup = testWorkflowType.Group,
                WorkflowTypeIcon = testWorkflowType.Icon,

                // Optionally set the default workflow to be mandatory (which means editors won't be able to remove it
                // via the back-office user interface).
                IsMandatory = true
            };

            // Retrieve the settings from the type.
            Dictionary<string, Core.Attributes.Setting> workflowTypeSettings = testWorkflowType.Settings();

            // Create a collection for the specific settings to be applied to the workflow.
            // Populate with the setting details from the type.
            var workflowSettings = new List<SettingWithValue>();
            foreach (KeyValuePair<string, Core.Attributes.Setting> setting in workflowTypeSettings)
            {
                Core.Attributes.Setting settingItem = setting.Value;

                var settingItemToAdd = new SettingWithValue
                {
                    Name = settingItem.Name,
                    Alias = settingItem.Alias,
                    Description = settingItem.Description,
                    Prevalues = settingItem.GetPreValues(),
                    View = IOHelper.ResolveUrl(settingItem.GetSettingView()),
                    Value = string.Empty
                };

                workflowSettings.Add(settingItemToAdd);
            }

            // For each settting, provide a value for the workflow instance (in this example, we only have one).
            SettingWithValue messageSetting = workflowSettings.SingleOrDefault(x => x.Alias == "Message");
            if (messageSetting != null)
            {
                messageSetting.Value = "A test log message";
            }

            // Apply the settings to the workflow.
            defaultWorkflow.Settings = workflowSettings;

            // Associate the workflow with the appropriate form submission event.
            form.FormWorkflows.OnSubmit.Add(defaultWorkflow);
        }
    }
}
```

Finally, to register the custom implementation in place of the default one:

```C#
using Umbraco.Core;
using Umbraco.Core.Composing;
using Umbraco.Forms.Web.Behaviors;

namespace MyNamespace
{
    public class TestSiteComposer : IUserComposer
    {
        public void Compose(Composition composition)
        {
            // Replace the default behavior for associating workflows with a custom implementation.
            composition.RegisterUnique<IApplyDefaultWorkflowsBehavior, CustomApplyDefaultWorkflowsBehavior>();
        }
    }
}
```

## Setting a Mandatory Default Workflow

When adding a default workflow in code, it's possible to make it mandatory, which will prevent editors from removing it from a form.

You can see this in the example above, where the `IsMandatory` property of the created `FormWorkflowWithTypeSettings` instance is set to `true`.

