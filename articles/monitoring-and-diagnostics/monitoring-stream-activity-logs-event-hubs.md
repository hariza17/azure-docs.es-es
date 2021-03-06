---
title: "Transmisión del registro de actividades de Azure a Event Hubs | Microsoft Docs"
description: Aprenda a transmitir el registro de actividad de Azure a Event Hubs.
author: johnkemnetz
manager: orenr
editor: 
services: monitoring-and-diagnostics
documentationcenter: monitoring-and-diagnostics
ms.assetid: ec4c2d2c-8907-484f-a910-712403a06829
ms.service: monitoring-and-diagnostics
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 01/30/2018
ms.author: johnkem
ms.openlocfilehash: c3c7ffe00263b8f76d89aa8d15fe2d502538527d
ms.sourcegitcommit: 9d317dabf4a5cca13308c50a10349af0e72e1b7e
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 02/01/2018
---
# <a name="stream-the-azure-activity-log-to-event-hubs"></a>Transmisión del registro de actividad de Azure a Event Hubs
El [**registro de actividades de Azure**](monitoring-overview-activity-logs.md) se puede transmitir casi en tiempo real a cualquier aplicación mediante la opción Exportar integrada en el portal o habilitando el identificador de regla de Service Bus en un perfil de registro por medio de los cmdlets de Azure PowerShell o la CLI de Azure.

## <a name="what-you-can-do-with-the-activity-log-and-event-hubs"></a>Qué se puede hacer con el registro de actividad y Event Hubs
Estas son solo algunas formas de usar la funcionalidad de streaming para el registro de actividad:

* **Transmisión a sistemas de registro y telemetría de terceros**: con el tiempo, el streaming de Event Hubs se convertirá en el mecanismo para canalizar el registro de actividad a sistemas de información de seguridad y administración de eventos (SIEM) y soluciones de análisis de registro de terceros.
* **Creación de una plataforma personalizada de registro y telemetría** : si ya tiene una plataforma de telemetría personalizada o está pensando en crear una, la gran escalabilidad en cuanto a la suscripción y la publicación de Event Hubs permite introducir el registro de actividad de manera flexible. [Consulte la guía de Dan Rosanova para usar Event Hubs en una plataforma de telemetría de escala global aquí.](https://azure.microsoft.com/documentation/videos/build-2015-designing-and-sizing-a-global-scale-telemetry-platform-on-azure-event-Hubs/)

## <a name="enable-streaming-of-the-activity-log"></a>Habilitación del streaming del registro de actividad
Puede habilitar el streaming del registro de actividad mediante programación o a través del portal. En cualquier caso, puede seleccionar un espacio de nombres de Service Bus y una directiva de acceso compartido para dicho espacio de nombres. Además, se crea un Centro de eventos en ese espacio de nombres cuando se produce el primer nuevo evento del registro de actividades. Si no tiene un espacio de nombres de Service Bus, primero debe crear uno. Si anteriormente se han transmitido eventos del registro de actividades para este espacio de nombres de Service Bus, se reutilizará el Centro de eventos que se creó anteriormente. La directiva de acceso compartido define los permisos que tiene el mecanismo de transmisión. En la actualidad, para transmitir a un centro de Event Hubs, se necesitan permisos de **administración**, **envío** y **escucha**. Puede crear o modificar las directivas de acceso compartido del espacio de nombres de Service Bus en Azure Portal en la pestaña "Configurar" del espacio de nombres de Service Bus. Para actualizar el perfil de registro del registro de actividad para incluir la transmisión, el cliente debe tener el permiso ListKey en la regla de autorización de Service Bus.

El espacio de nombres del centro de eventos o el bus de servicio no tiene que estar en la misma suscripción que la que emite los registros, siempre que el usuario que configura la configuración tenga acceso RBAC adecuado a ambas suscripciones.

### <a name="via-azure-portal"></a>Mediante el Portal de Azure
1. Vaya a la hoja de **registro de actividad** mediante la búsqueda de todos los servicios en el menú del lado izquierdo del portal.
   
    ![Ir al registro de actividad en el portal](./media/monitoring-stream-activity-logs-event-hubs/activity.png)
2. Haga clic en el botón **Exportar** en la parte superior de la hoja de registro de actividad.
   
    ![Botón Exportar en el portal](./media/monitoring-stream-activity-logs-event-hubs/export.png)
3. En la hoja que aparece, puede seleccionar las regiones para las que transmitir eventos y el espacio de nombres de Service Bus donde desea que se cree un centro de eventos para transmitirlos. Seleccione **Todas las regiones**.
   
    ![Exportar en hoja de registro de actividad](./media/monitoring-stream-activity-logs-event-hubs/export-audit.png)
4. Haga clic en **Guardar** para guardar la configuración. La configuración se aplica inmediatamente a la suscripción.
5. Si tiene varias suscripciones, debe repetir esta acción y enviar todos los datos al mismo centro de datos.

### <a name="via-powershell-cmdlets"></a>Mediante cmdlets de PowerShell
Si ya existe un perfil de registro, primero debe quitarlo.

1. Use `Get-AzureRmLogProfile` para identificar si existe un perfil de registro.
2. Si es así, use `Remove-AzureRmLogProfile` para quitarlo.
3. Use `Set-AzureRmLogProfile` para crear un perfil:

```powershell

Add-AzureRmLogProfile -Name my_log_profile -serviceBusRuleId /subscriptions/s1/resourceGroups/Default-ServiceBus-EastUS/providers/Microsoft.ServiceBus/namespaces/mytestSB/authorizationrules/RootManageSharedAccessKey -Locations global,westus,eastus -RetentionInDays 90 -Categories Write,Delete,Action

```

El identificador de regla de Service Bus es una cadena con este formato: por ejemplo, {Id. de recurso de Service Bus}/authorizationrules/{nombre de clave}. 

### <a name="via-azure-cli"></a>Mediante la CLI de Azure
Si ya existe un perfil de registro, primero debe quitarlo.

1. Use `azure insights logprofile list` para identificar si existe un perfil de registro.
2. Si es así, use `azure insights logprofile delete` para quitarlo.
3. Use `azure insights logprofile add` para crear un perfil:

```azurecli-interactive
azure insights logprofile add --name my_log_profile --storageId /subscriptions/s1/resourceGroups/insights-integration/providers/Microsoft.Storage/storageAccounts/my_storage --serviceBusRuleId /subscriptions/s1/resourceGroups/Default-ServiceBus-EastUS/providers/Microsoft.ServiceBus/namespaces/mytestSB/authorizationrules/RootManageSharedAccessKey --locations global,westus,eastus,northeurope --retentionInDays 90 –categories Write,Delete,Action
```

El identificador de regla de Service Bus es una cadena con este formato: `{service bus resource ID}/authorizationrules/{key name}`.

## <a name="how-do-i-consume-the-log-data-from-event-hubs"></a>¿Cómo se consumen los datos de registro procedentes de Event Hubs?
[El esquema para el registro de actividades está disponible aquí](monitoring-overview-activity-logs.md). Cada evento está en una matriz de blobs JSON denominados "registros".

## <a name="next-steps"></a>Pasos siguientes
* [(Archivado del registro de actividades en una cuenta de almacenamiento)](monitoring-archive-activity-log.md)
* [Lea la información general sobre el registro de actividades de Azure](monitoring-overview-activity-logs.md)
* [Configure una alerta basada en un evento del registro de actividades](insights-auditlog-to-webhook-email.md)

