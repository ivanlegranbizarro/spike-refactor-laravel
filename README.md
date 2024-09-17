# Implementación de Model Binding en Laravel

Este documento describe los pasos y cambios necesarios para implementar Model Binding en un proyecto Laravel.

## Cambios en las Rutas

Lo primero es modificar la variable de interpolación en las rutas. Las rutas deben incluir el nombre de la entidad en minúsculas.

### Ruta Antigua:
```php
Route::get('student/{studentId}/resume/detail', StudentDetailController::class)->name('student.details');
```

### Nueva Ruta:
```php
Route::get('student/{student}/detail', StudentDetailController::class)->name('student.details');
```

## Impacto en los Tests

Una vez implementada la inyección del modelo en los controllers, es probable que los tests fallen inicialmente. Esto se debe a que el mensaje que devuelve Laravel por defecto es diferente al que se había creado previamente.

### Ejemplo de Error en Test:
```php
Failed asserting that an array has the subset Array &0 [
    'message' => 'No s'ha trobat cap estudiant amb aquest ID: 12345',
].
   array (
  -  'message' => 'No s\'ha trobat cap estudiant amb aquest ID: 12345',
  +  'message' => 'No query results for model [App\\Models\\Student] 12345',
   )
```

## Cambios en el Controller

El controller se simplifica significativamente con Model Binding. Ya no es necesario llamar a métodos de servicio en la mayoría de los casos.

### Controller Antiguo:
```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\api\Student;

use Exception;
use App\Http\Controllers\Controller;
use App\Service\Student\StudentDetailService;
use Illuminate\Http\JsonResponse;
use App\Exceptions\StudentNotFoundException;
use App\Exceptions\ResumeNotFoundException;

class StudentDetailController extends Controller
{
    private StudentDetailService $studentDetailsService;

    public function __construct(StudentDetailService $studentDetailsService)
    {
        $this->studentDetailsService = $studentDetailsService;
    }

    public function __invoke($studentId):JsonResponse
    {
        try {
            $service = $this->studentDetailsService->execute($studentId);
            return response()->json(['data'=> [$service]]);
        } catch (StudentNotFoundException | ResumeNotFoundException $e) {
            return response()->json(['message' => $e->getMessage()], $e->getCode());
        } catch (Exception $e) {
            return response()->json(['message' => $e->getMessage()], $e->getCode() ?: 500);
        }
    }
}
```

### Nuevo Controller:
```php
<?php

namespace App\Http\Controllers\api\Student;

use App\Http\Controllers\Controller;
use App\Models\Student;
use Illuminate\Http\JsonResponse;

class StudentDetailController extends Controller
{
    public function __invoke(Student $student): JsonResponse
    {
        return response()->json($student);
    }
}
```

## Cambios en los Tests

Los tests deben adaptarse para reflejar el nuevo comportamiento del controller y el uso de Model Binding.

### Test Antiguo:
```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Controller\Student;

use Illuminate\Foundation\Testing\DatabaseTransactions;
use Tests\TestCase;
use App\Http\Controllers\api\Student\StudentDetailController;
use App\Service\Student\StudentDetailService;
use App\Models\Student;

class StudentDetailControllerTest extends TestCase
{
    use DatabaseTransactions;

    protected $student;
    protected $resume;

    protected function setUp(): void
    {
        parent::setUp();

        $this->student = Student::factory()->create();
        $this->resume = $this->student->resume()->create();
    }

    public function testStudentDetailsAreFound(): void
    {
        $studentDetailService = $this->createMock(StudentDetailService::class);

        $student = $this->student;

        $studentId = $student->id;

        $studentDetail = $this->resume;

        $studentDetailService->expects($this->once())
                              ->method('execute')
                              ->with($studentId)
                              ->willReturn($studentDetail->toArray());

        $this->app->instance(StudentDetailService::class, $studentDetailService);

        $response = $this->get(route('student.details', ['studentId' => $studentId]));

        $response->assertStatus(200);

        $response->assertJsonStructure();
    }
}
```

### Nuevo Test:
```php
<?php

namespace Tests\Feature\Controller\Student;

use Illuminate\Foundation\Testing\DatabaseTransactions;
use Tests\TestCase;
use App\Models\Student;

class StudentDetailControllerTest extends TestCase
{
    use DatabaseTransactions;

    protected $student;
    protected $resume;

    protected function setUp(): void
    {
        parent::setUp();

        $this->student = Student::factory()->create();
        $this->resume = $this->student->resume()->create();
    }

    public function testStudentDetailsAreFound(): void
    {
        $response = $this->get(route('student.details', ['student' => $this->student]));

        $response->assertStatus(200);

        $response->assertJson([
            'id' => $this->student->id,
        ]);
    }

    public function testStudentDetailsIsNotFound(): void
    {
        $nonExistentStudentId = 12345;

        $response = $this->get(route('student.details', ['student' => $nonExistentStudentId]));

        $response->assertStatus(404);

        $response->assertJson(['message' => 'No query results for model [App\\Models\\Student] ' . $nonExistentStudentId]);
    }
}
```

## Conclusión

La implementación de Model Binding en Laravel simplifica significativamente el código del controller y los tests asociados. Elimina la necesidad de servicios intermedios para la mayoría de las operaciones básicas de CRUD, permitiendo una interacción más directa con los modelos de Eloquent.

Es importante actualizar los mensajes de error y las expectativas en los tests para reflejar los nuevos comportamientos proporcionados por Laravel cuando se utiliza Model Binding.
